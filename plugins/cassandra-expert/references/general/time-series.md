# Time Series Data Modeling in Cassandra

## Core Principles

### Primary Key Design
- Use compound partition keys with time-based bucketing
- Keep partitions under 100MB
- Use `timeuuid` for timestamps (ordering + uniqueness)
- Structure: `PRIMARY KEY((partition_key, time_bucket), timeuuid)`

## Bucketing Strategies

### Strategy 1: Time-Based Partitioning

Prevent unbounded partition growth by adding time windows to partition keys:

```cql
CREATE TABLE raw_data_by_day (
    sensor text,
    day text,
    ts timeuuid,
    reading int,
    PRIMARY KEY((sensor, day), ts)
) WITH CLUSTERING ORDER BY (ts DESC);
```

**When to use:**
- Predictable data volumes per time window
- Choose window size (hour, day, week, month) based on ingest rate
- Leave headroom for growth

**Benefits:**
- Distributes reads across cluster
- Queries can be parallelized
- Natural archiving boundaries

### Strategy 2: Write Distribution (Scatter-Gather)

Fan out writes across multiple partitions for high-volume scenarios:

```cql
CREATE TABLE tweet_stream (
    account text,
    day text,
    bucket int,
    ts timeuuid,
    message text,
    PRIMARY KEY((account, day, bucket), ts)
) WITH CLUSTERING ORDER BY (ts DESC);
```

**When to use:**
- Highly variable ingest rates
- Need to scale beyond single-node write capacity

**Trade-offs:**
- Reads require scatter-gather across all buckets
- More complex pagination
- Too many buckets = many small reads; too few = large partitions

**Reading strategy:**
- Query all buckets in parallel
- Merge results client-side

## Table-Per-Time-Window Pattern

Create separate tables for time windows (monthly or yearly) rather than relying solely on partition-level bucketing.

```cql
CREATE TABLE sensor_data_2025_01 (
    sensor_id text,
    day text,
    ts timeuuid,
    value double,
    PRIMARY KEY((sensor_id, day), ts)
) WITH CLUSTERING ORDER BY (ts DESC)
  AND gc_grace_seconds = 60;
```

### Benefits
- Reduced compaction overhead - smaller tables compact faster
- Reduced repair overhead - operates on smaller datasets
- Drop entire tables when data expires - instant deletion
- No tombstone overhead from deletes
- Easy archiving before deletion.  Either copy SSTables or use Analytics project.

### Lifecycle Management

Application logic handles:

1. Writing to current month's table
2. Reading across multiple tables for spanning queries
3. Archiving old tables before retention expires
4. Dropping tables when retention period ends

### Table-Per-Window vs TTL

| Approach | Pros                                                  | Cons |
|----------|-------------------------------------------------------|------|
| Table-per-window | Instant deletion via DROP, no tombstones, can archive | Application complexity |
| TTL | Simpler application code                              | Depends on compaction |

## Compaction Strategy

### Cassandra 5.0+

Use UCS for time series workloads. Choose scaling parameters based on your workload:

| Parameter | Behavior | Trade-off |
|-----------|----------|-----------|
| `T4` | Tiered, 4 SSTables per tier | Less compaction I/O, but more SSTables per read |
| `L10` | Leveled, 10 SSTables per level | More compaction I/O, but fewer SSTables per read |

**Write-heavy time series** (most ingest scenarios):
```cql
WITH compaction = {
    'class': 'UnifiedCompactionStrategy',
    'scaling_parameters': 'T4'
};
```

**Read-heavy time series** (frequent queries on recent data):
```cql
WITH compaction = {
    'class': 'UnifiedCompactionStrategy',
    'scaling_parameters': 'L10'
};
```

The right choice depends on your workload and hardware.

### Cassandra 4.x and Earlier

Using TWCS for immutable time series:

```cql
WITH compaction = {
    'class': 'TimeWindowCompactionStrategy',
    'compaction_window_size': 7,
    'compaction_window_unit': 'DAYS'
};
```

Exact sizing of the window depends on your query patterns.  Smaller windows = less compaction overhead, but more SSTables per read.

## gc_grace_seconds for Time Series

For immutable time series data with no explicit deletes:

```cql
AND gc_grace_seconds = 60;
```

Low gc_grace is safe when:
- Data is immutable (no updates/deletes)
- TTL handles expiration
- No explicit DELETE statements

## Avoiding Explicit Deletes

**Do not use DELETE on time series data:**
- Creates tombstones
- With TWCS: tombstones end up in different windows than data
- Space never reclaimed efficiently

**Instead:**
- Use table drops (table-per-window pattern)
- Use TTL at write time 


## Read Patterns

### Multi-Table Queries
Query each table in parallel, merge client-side.

### Multi-Day Queries (Same Table)
Issue one query per time bucket, parallelize with driver.

### Driver Support
- Python: `execute_concurrent_with_args()`
- Java: async execution with CompletableFuture
- Distributes work across entire cluster

## Complete Examples

### Cassandra 5.0+ with Table-Per-Window

```cql
CREATE TABLE sensor_data_2025_01 (
    sensor_id text,
    day text,
    ts timeuuid,
    value double,
    PRIMARY KEY((sensor_id, day), ts)
) WITH CLUSTERING ORDER BY (ts DESC)
  AND compaction = {
      'class': 'UnifiedCompactionStrategy',
      'scaling_parameters': 'T4'
  }
  AND gc_grace_seconds = 60;
```

### Cassandra 5.0+ with TTL

```cql
CREATE TABLE sensor_data (
    sensor_id text,
    day text,
    ts timeuuid,
    value double,
    PRIMARY KEY((sensor_id, day), ts)
) WITH CLUSTERING ORDER BY (ts DESC)
  AND compaction = {
      'class': 'UnifiedCompactionStrategy',
      'scaling_parameters': 'T4'
  }
  AND default_time_to_live = 7776000  -- 90 days
  AND gc_grace_seconds = 60;
```

### Cassandra 4.x with TWCS

```cql
CREATE TABLE sensor_data (
    sensor_id text,
    day text,
    ts timeuuid,
    value double,
    PRIMARY KEY((sensor_id, day), ts)
) WITH CLUSTERING ORDER BY (ts DESC)
  AND compaction = {
      'class': 'TimeWindowCompactionStrategy',
      'compaction_window_size': 1,
      'compaction_window_unit': 'DAYS',
      'unchecked_tombstone_compaction': true
  }
  AND default_time_to_live = 7776000
  AND gc_grace_seconds = 60;
```

When an sstable is written a histogram with the tombstone expiry times is created and this is used to try to find
SSTables with very many tombstones and run single sstable compaction on that sstable in hope of being able to 
drop tombstones in that sstable. Before starting this it is also checked how likely it is that any tombstones will 
actually will be able to be dropped how much this sstable overlaps with other SSTables. 
To avoid most of these checks the compaction option `unchecked_tombstone_compaction` can be enabled.


## Common Pitfalls

### Unbounded Partitions
Always include time bucket in partition key.

### Explicit Deletes
Use TTL or table drops instead.

### TWCS Timestamp Mixing
Hints, read repairs can mix timestamps across windows. UCS handles this better.

### Write Hot Spots
Single partition writes create hot spots. Use bucketing to spread writes.
