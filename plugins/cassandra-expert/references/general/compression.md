# Compression in Cassandra

## Overview

Compression is often overlooked but significantly impacts performance and node density. The default 64KB chunk length in older version is rarely optimal for read-heavy workloads.

## How Compression Works

1. Data is divided into fixed-size chunks (default: 64KB / 16KB as of Cassandra 4.0, CASSANDRA-13241)
2. Each chunk is compressed independently
3. A compression offset map tracks chunk locations (uses off-heap memory)
4. During reads, only necessary chunks are decompressed

## Chunk Size: The Critical Setting

### The Problem with 64KB Chunks

Tables created in older Cassandra versions (pre 4.0) used 64KB as the default chunk size. This causes:

- Reading and decompressing 64KB to retrieve small amounts of data
- Wasted CPU and disk I/O
- Higher read latency
- Earlier saturation under load
- Poor LWT and Counter performance

### 4KB Chunks for Read-Heavy Workloads

```cql
ALTER TABLE keyspace.table WITH compression = {
    'sstable_compression': 'LZ4Compressor',
    'chunk_length_kb': 4
};
```

Small increase in total disk space used, significantly higher throughput.

### Trade-offs

| Chunk Size | Read Performance                              | Compression Ratio | Memory Usage |
|------------|-----------------------------------------------|-------------------|--------------|
| 4KB | Best for point lookups                        | Slightly lower | 16x more than 64KB |
| 16KB | Good balance                                  | Good | 4x more than 64KB |
| 64KB | Not generally great. Better for large slices. | Best | Baseline |

**Memory consideration:** 4KB chunks use 16x more off-heap memory for compression metadata. A 20TB node might use 4-5GB for offset maps.

## Algorithm Selection

### LZ4 (Default)
Best for most workloads - good compression with minimal CPU overhead.

```cql
'sstable_compression': 'LZ4Compressor'
```

### Zstd (Cassandra 4.0+)
Better compression ratio, more CPU intensive. Good for cold/archival data.

```cql
'sstable_compression': 'ZstdCompressor'
```

### No Compression
Consider for already-compressed data (images, videos) or extreme CPU constraints.

```cql
WITH compression = {'enabled': false};
```

## Workload-Specific Configurations

### Read-Heavy Tables

Tables using counters and LWT are _at least_ 50% reads due to read before write.

```cql
ALTER TABLE keyspace.table WITH compression = {
    'sstable_compression': 'LZ4Compressor',
    'chunk_length_kb': 4
};
```

### Write-Heavy Tables

Generally still better to use 16KB over 64KB unless you know you will _rarely_ read individual rows.

```cql
ALTER TABLE keyspace.table WITH compression = {
    'sstable_compression': 'LZ4Compressor',
    'chunk_length_kb': 16
};
```

### Archival/Cold Tables

```cql
ALTER TABLE keyspace.table WITH compression = {
    'sstable_compression': 'ZstdCompressor',
    'chunk_length_kb': 16,
    'compression_level': 3
};
```

### Counter Tables

Counters perform read-before-write, so optimize for reads:

```cql
ALTER TABLE keyspace.counters WITH compression = {
    'sstable_compression': 'LZ4Compressor',
    'chunk_length_kb': 4
};
```

## Applying Changes

Changes only affect new SSTables. To rewrite existing data:

```bash
nodetool upgradesstables -a keyspace table
```

**Warning:** This rewrites all SSTables - schedule during low traffic.

## Monitoring

### Check Compression Ratio

```bash
nodetool tablestats keyspace.table
# Look for "SSTable Compression Ratio" (lower = better, typically 0.3-0.7)
```

### Check Off-Heap Memory Usage

```bash
nodetool tablestats keyspace.table | grep "Compression metadata off heap"
```

### Assess Read Performance Impact

Using built in tracing:

```cql
TRACING ON;
SELECT * FROM keyspace.table WHERE pk = 'value' LIMIT 10;
TRACING OFF;
-- Check time in "Decompressing" phase
```

## Memory Planning

When using 4KB chunks, account for increased off-heap memory:

```yaml
# Ensure sufficient native memory
# 4KB chunks: ~16x memory vs 64KB for offset maps
# Example: 20TB data might use 4-5GB for compression metadata
```
