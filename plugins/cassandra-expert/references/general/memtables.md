# Memtables in Cassandra

## Overview

Memtables are in-memory structures where Cassandra buffers writes before flushing to SSTables. In Cassandra 5.0+, Trie memtables provide significant performance improvements.

Combined with offheap memtables I have seen a 250% improvement in write throughput when low latency is required. Your mileage may vary.

## Trie Memtables (Cassandra 5.0+)

### Why Trie Memtables Matter

Trie memtables replace the traditional SkipList implementation with a more efficient trie-based structure:

- Significantly reduced allocations
- Lower GC pause frequency
- Shorter GC pause duration
- Improved write throughput under load
- Better memory efficiency

### Enabling Trie Memtables

Trie memtables require explicit configuration in `cassandra.yaml`:

```yaml
memtable:
  configurations:
    skiplist:
      class_name: SkipListMemtable
    trie:
      class_name: TrieMemtable
    default:
      inherits: trie
```

**Important:** This is not enabled by default. You must add this configuration.

### Per-Table Override

You can use different memtable types per table:

```cql
-- Use SkipList for specific table
CREATE TABLE keyspace.table (...)
WITH memtable = 'skiplist';

-- Use Trie (if default is skiplist)
CREATE TABLE keyspace.table (...)
WITH memtable = 'trie';
```

## Monitoring Memtables

### Check Memtable Statistics

```bash
nodetool tablestats keyspace.table
# Look for memtable-related metrics
```

### Monitor Flush Activity

```bash
nodetool tpstats | grep MemtableFlushWriter
# Watch for pending/blocked tasks
```

### Warning Signs

| Symptom | Likely Cause |
|---------|--------------|
| Flush storms (many rapid flushes) | Memtable space too small for workload |
| Frequent old gen GC | Not using Trie memtables on 5.0+ |
| Blocked MemtableFlushWriter | Disk I/O bottleneck or too many tables |
| Many small SSTables | Memtable space fragmented across too many tables |

## Many Tables Consideration

With many tables (50+), memtable memory fragments across all active tables:

- Each table gets smaller portion of total space
- Results in smaller, more frequent flushes
- Creates more SSTables
- Increases compaction overhead

**Solutions:**
- Consolidate tables where data model allows
- Drop unused tables
- Ensure adequate memtable space for table count

## Interaction with Other Features

### With Off-Heap Storage

Trie memtables work optimally with off-heap storage, reducing heap pressure further.

### With Compaction

Larger memtable flushes create fewer, larger SSTables, reducing compaction overhead.

### With GC

Trie memtables significantly reduce allocation rate, which:
- Reduces young gen GC frequency
- Reduces promotion to old gen
- Shortens GC pause times

## Migration to Trie Memtables

When upgrading to Cassandra 5.0+:

1. Add the memtable configuration to `cassandra.yaml`
2. Perform rolling restart
3. Monitor GC behavior
4. Monitor write latency and throughput
