# Cassandra 5.0 JVM Options Recommendations

## Java Version

**Cassandra 5.0 requires Java 17+.**

JVM options are configured in:
- `jvm-server.options` - common settings
- `jvm17-server.options` - Java 17 specific settings

## Garbage Collector Selection

### G1GC (Recommended Default)

G1GC is the default and works well for most workloads. It's easier to configure and mostly hands-off.

```
-XX:+UseG1GC
-XX:+ParallelRefProcEnabled
-XX:MaxGCPauseMillis=200
-XX:G1HeapRegionSize=16m
-XX:MaxTenuringThreshold=4
-XX:G1NewSizePercent=50
-XX:InitiatingHeapOccupancyPercent=70
```

**Key tuning point: Use a larger new generation (50%+).**

Cassandra is allocation-heavy. High allocations lead to high promotions to old gen. A larger new gen:
- Allows more time between GC cycles
- Reduces promotion rate
- Can reduce both pause duration and frequency

Testing shows heaps from 16-31GB work well with 50% new gen. Some clusters run 75% new gen on 31GB heaps with improvements in both pause duration and frequency.

### Shenandoah GC

Shenandoah delivers sub-millisecond pause times by performing garbage collection concurrently with the application.

```
-XX:+UseShenandoahGC
-XX:+UnlockExperimentalVMOptions
-XX:ShenandoahGCHeuristics=adaptive
```

**Benefits:**
- P99 latency reductions of 40-60% reported in production
- Pause times not proportional to heap size
- ~10ms pause targets achievable

**When to use Shenandoah:**
- Latency-sensitive workloads
- Moderate, steady write rates
- When consistent low latency matters more than peak throughput

**When NOT to use Shenandoah:**

Shenandoah can waste significant CPU time with compaction-heavy workloads:
- Spark jobs doing large bulk writes
- Analytics workloads with heavy write amplification
- Any workload that generates sustained high allocation rates

For bulk/analytics workloads, consider the Cassandra Analytics project to offload these operations instead of running them through the main Cassandra process.

**Note:** The "experimental" flag is required but Shenandoah is production-ready on Java 17. The flag just enables tuning options that may change between JVM versions.

## Heap Sizing

### General Guidelines

```
-Xms31G
-Xmx31G
```

- Set `-Xms` and `-Xmx` to the same value to avoid runtime resizing
- G1GC works well with larger heaps (up to 31GB common, up to 64GB possible)
- Leave room for off-heap structures and OS page cache

### Heap Size Considerations

- **Minimum**: G1 struggles with heaps that are too small
- **Sweet spot**: ~30x your allocation rate
- **Maximum**: Auto-generated default is half RAM, capped at 31GB for G1
- **Off-heap**: Account for memtable off-heap space, bloom filters, compression metadata

### With Off-Heap Trie Memtables

Recent testing with off-heap trie memtables showed GC pause times low enough to achieve 2-4x throughput improvement when capping p99 latency at 20ms.

## Alternative: Shenandoah Configuration

```
# Heap
-Xms31G
-Xmx31G

# Shenandoah GC
-XX:+UseShenandoahGC
-XX:+UnlockExperimentalVMOptions
-XX:ShenandoahGCHeuristics=adaptive

# GC Logging
-Xlog:gc*:file=/var/log/cassandra/gc.log:time,uptime,level,tags:filecount=10,filesize=10M

# Performance
-XX:+AlwaysPreTouch
```

## Future: JDK 21 and Generational ZGC

JDK 21 support is in progress for Cassandra. Generational ZGC is expected to provide further improvements. Watch for this in future Cassandra releases.
