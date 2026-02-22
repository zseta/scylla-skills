---
name: optimize
description: Performance optimization for Apache Cassandra clusters. Use when tuning configuration, improving throughput, reducing latency, or optimizing resource usage.
argument-hint: [current config, metrics, or performance goal]
user-invocable: true
---

# Cassandra Performance Optimization

You are an expert Cassandra performance engineer focused on configuration tuning and optimization.

## System Settings

**Read-ahead is critical:**

The most critical setting that affects performance and cost is read-ahead. Read ahead, especially with Cassandra 5.0+, offers no benefits, and is one of the worst settings you can have enabled.

Check with: `sudo blockdev --report`

Disable or minimize read-ahead for Cassandra data volumes.

## cassandra.yaml Critical Settings

### Token Configuration
- `num_tokens` - 1 or max of 4. Higher is always worse.
  - More tokens = more data movement during scaling
  - More tokens = more overhead in token calculations

### Thread Pool Sizing
- `concurrent_reads` / `concurrent_writes` - thread pool sizing
  - Default is often reasonable, but monitor saturation
  - Increase if threads are consistently saturated with low CPU usage

### Memory Configuration
- `memtable_heap_space_in_mb` (or `memtable_heap_space`)
- `memtable_offheap_space_in_mb` (or `memtable_offheap_space`)
  - Off-heap delivers better performance
  - Especially in Cassandra 5.0+ where Trie memtables are available
  - Use `class_name: TrieMemtable` for best performance

### Compaction Settings
- `compaction_throughput_mb_per_sec` - compaction throttling
  - Compaction uses system resources and is allocation heavy, leading to GC pauses
  - Has negative impact on page cache
  - See [JIRA](https://issues.apache.org/jira/browse/CASSANDRA-19987)
  - See [Jon's blog on compaction throughput](https://rustyrazorblade.com/post/2025/04-compaction-throughput/)

### Cache Settings
- `row_cache_size_in_mb`: Keep disabled. Row cache is rarely beneficial and often harmful.
- `key_cache`: Generally useful, leave enabled

### Durability Settings
- `commitlog_sync_period_in_ms`: 10s is outdated on modern hardware. 1 second is more practical and reduces data loss potential.

## JVM Settings

Located in `jvm.options` or `jvm11-server.options`:

### Garbage Collection
- G1 is easier to configure, mostly hands off
- Recommendation: use a larger new gen (50%+) than configs historically used
  - Cassandra is allocation heavy
  - High allocations means high promotions
  - Larger new gen allows more time between GC cycles
  - This reduces the promotion rate

### Heap Sizing
- Match to workload and available memory
- Leave room for off-heap structures and OS page cache
- Monitor GC logs to validate sizing

## Per-Table Settings

### Compaction Strategy

**Cassandra 5.0+:** Jon recommends UCS (Unified Compaction Strategy) for all tables.

**STCS should NEVER be used:**
- Especially with 4.0+
- Makes scaling node density far more challenging / impossible
- See [Jon's Blog on Compaction Strategies](https://rustyrazorblade.com/post/2025/07-compaction-strategies-and-performance/)

**Strategy Selection:**
- UCS: Best for 5.0+, handles all workloads well
- LCS: Good for read-heavy workloads (pre-5.0)
- TWCS: Time-series data with TTL

### Compression Settings
- Older versions used 64KB `chunk_length_in_kb`
- This leads to terrible performance
- Larger chunk length means decoding more data on each read
- Data is rarely sent back entirely - wasted CPU and disk I/O causing early saturation
- Consider smaller chunk sizes for read-heavy workloads

### Other Table Settings
- `bloom_filter_fp_chance` - lower = more memory, fewer false positives
- `gc_grace_seconds` - align with your repair schedule and TTL
- TTL - use when data has natural expiration

## Consistency Level Trade-offs

| Level | Reads | Writes | Trade-off |
|-------|-------|--------|-----------|
| ONE | Fastest | Fastest | Risk of stale reads |
| QUORUM | Balanced | Balanced | Strong consistency |
| LOCAL_QUORUM | DC-local | DC-local | Best for multi-DC |
| ALL | Slowest | Slowest | Maximum consistency |

## Cross-Node Configuration Consistency

- Verify configuration is consistent across all nodes
- Check for drift from intended configuration
- Review recent configuration changes in change management
- Use configuration management tools (Ansible, Chef, Puppet)

## Cassandra 5.0 References

For detailed Cassandra 5.0 configuration guidance, read:
- `../../references/cassandra-5.0/notable-features.md` - UCS, Trie memtables, BTI, Zero-Copy Streaming
- `../../references/cassandra-5.0/cassandra-yaml.md` - Full cassandra.yaml recommendations
- `../../references/cassandra-5.0/jvm-options.md` - JVM and GC tuning (G1, Shenandoah)

## Optimization Checklist

1. **System Level**
   - [ ] Read-ahead disabled/minimized
   - [ ] Appropriate I/O scheduler (noop/none for SSD)
   - [ ] Swappiness set low (1)
   - [ ] Sufficient file descriptors

2. **JVM Level**
   - [ ] Appropriate heap size
   - [ ] G1GC with tuned new gen
   - [ ] GC logging enabled

3. **Cassandra Level**
   - [ ] num_tokens minimized (1-4)
   - [ ] Appropriate compaction strategy (UCS preferred)
   - [ ] Compression chunk size optimized
   - [ ] Off-heap memtables enabled (5.0+)

4. **Table Level**
   - [ ] Compaction strategy matches workload
   - [ ] TTL and gc_grace_seconds aligned
   - [ ] Appropriate bloom filter settings
