---
name: data-model
description: Data modeling and schema design for Apache Cassandra. Use when designing tables, choosing partition keys, modeling time-series data, or reviewing existing schemas.
argument-hint: [query patterns, use case, or schema to review]
---

# Cassandra Data Modeling

You are an expert Cassandra data modeler focused on query-driven schema design.

## Core Principles

### Query-First Design
- Start with the queries you need to support
- Design tables to satisfy each query pattern
- Denormalization is expected and necessary
- One table per query pattern is common

### Partition Key Selection
- Determines data distribution across nodes
- Must provide even distribution (avoid hot partitions)
- Should match your query's WHERE clause equality predicates
- Composite partition keys: `PRIMARY KEY ((col1, col2), col3)`

### Clustering Key Design
- Determines sort order within a partition
- Enables efficient range queries
- Order matters: `CLUSTERING ORDER BY (col DESC)`
- Support your query's ORDER BY and range predicates

## Partition Sizing Guidelines

**Target: Under 10MB per partition**

Jon's recommendation is to stay under 10MB per partition:
- If you're going to use multiple partitions anyway, keep them manageable
- You can't read all 10MB at once
- Pagination requires separate queries per page
- There's no downside to smaller partitions

**Warning Signs:**
- Partitions > 100MB - serious problem
- Partitions > 100K rows - review design
- Unbounded partition growth - add time bucketing

## Time-Series Data Modeling

**Pattern: Table per time period, partition per day**

```sql
CREATE TABLE events_2024_01 (
    sensor_id uuid,
    event_date date,
    event_time timestamp,
    value double,
    PRIMARY KEY ((sensor_id, event_date), event_time)
) WITH CLUSTERING ORDER BY (event_time DESC);
```

**Key Strategies:**
- Table per month/year for easy data lifecycle management
- Partition per day to bound partition size
- Clustering by time for efficient range queries
- TTL for automatic data expiration

Reference: [Time Series for Massive Scale](https://www.youtube.com/watch?v=Ysfi3V2KQtU)

## Common Patterns

### User Activity
```sql
CREATE TABLE user_activity (
    user_id uuid,
    activity_date date,
    activity_time timestamp,
    activity_type text,
    details map<text, text>,
    PRIMARY KEY ((user_id, activity_date), activity_time)
) WITH CLUSTERING ORDER BY (activity_time DESC);
```

### Lookup Table
```sql
CREATE TABLE users_by_email (
    email text,
    user_id uuid,
    PRIMARY KEY (email)
);
```

### Wide Rows with Bucketing
```sql
CREATE TABLE messages (
    conversation_id uuid,
    bucket int,  -- derived from message_time
    message_time timestamp,
    message_id uuid,
    content text,
    PRIMARY KEY ((conversation_id, bucket), message_time, message_id)
) WITH CLUSTERING ORDER BY (message_time DESC);
```

## Multi-Tenant Strategies

### Tenant in Partition Key
```sql
PRIMARY KEY ((tenant_id, entity_id), ...)
```
- Good isolation
- Easy to query within tenant
- Cross-tenant queries impossible (usually desired)

### Separate Keyspaces
- Maximum isolation
- Different replication per tenant possible
- More operational overhead

## Anti-Patterns to Avoid

### ALLOW FILTERING
- Never use in production queries
- Indicates wrong data model
- Causes full table scans

### Secondary Indexes on High-Cardinality Columns
- Poor performance at scale
- Consider denormalization instead
- SAI (5.0+) improves this but still not ideal for high cardinality
- SAI will deliver decent performance in small clusters without a partition key in the query, but scales linearly with the total number of SSTables. **Always use a partition key** 

### Unbounded Partitions
- Always add time bucketing for growing data
- Use TTL when applicable
- Monitor partition sizes

### Read-Before-Write
- Cassandra is optimized for writes
- Read-modify-write patterns are expensive
- Design to avoid when possible

### Large IN Clauses
- Each value is a separate query internally
- Prefer multiple single-value queries
- Or redesign the data model

### Multi-Partition Batches
- Don't batch across partitions
- Use unlogged batches only for same-partition atomicity
- Let the driver handle multi-partition writes

### Materialized Views
- Materialized views require careful planning, or they can completely destroy your cluster
- They can't be reliably repaired.
- Jon **strongly** advises you do not use them.

## Schema Review Checklist

When reviewing a schema, check:

1. **Partition Key**
   - [ ] Provides even data distribution?
   - [ ] Matches query WHERE clause?
   - [ ] Partition size bounded?

2. **Clustering Key**
   - [ ] Supports required sort order?
   - [ ] Enables needed range queries?
   - [ ] Order matches query patterns?

3. **Data Types**
   - [ ] Appropriate types for data?
   - [ ] Collections used appropriately?
   - [ ] Frozen vs non-frozen considered?

4. **Query Support**
   - [ ] Each query has a supporting table?
   - [ ] No ALLOW FILTERING needed?
   - [ ] Consistency requirements met?

## Guidelines

When designing schemas:
1. List all query patterns first
2. Design one table per query pattern
3. Denormalize data across tables
4. Bound partition sizes with bucketing
5. Consider data lifecycle (TTL, table rotation)
6. Test with realistic data volumes
