# Compaction in Cassandra

## Overview

Compaction merges SSTables, removes deleted and overwritten data, and organizes data for efficient reads. 

Strategy selection directly impacts node density, operational costs, and performance.

## Strategy Selection

### Decision Tree

```
Is your Cassandra version 5.0 or later?
├── YES → Use UCS (for all workloads)
└── NO (4.x or earlier)
    └── Is this a time series workload?
        ├── YES → Is data immutable with TTL?
        │   ├── YES → Use TWCS
        │   └── NO → Use LCS with table bucketing by date
        └── NO → Is this a write-intensive workload with overwrites?
            ├── YES → Consider STCS (rare case)
            └── NO → Use LCS (best general-purpose choice)
```

### Cassandra 5.0+: Use UCS for Everything

**UnifiedCompactionStrategy (UCS)** replaces all legacy strategies:

- Adapts to workload patterns automatically
- Controlled SSTable sizes (default 1GB)
- Better streaming performance (Zero Copy eligible)
- Reduced write amplification

### Cassandra 4.x and 3.x

| Strategy | Use Case | Trade-offs |
|----------|----------|------------|
| **LCS** | General purpose, read-heavy | Higher write amp, consistent reads |
| **TWCS** | Immutable time series with TTL | Large SSTables prevent ZCS |
| **STCS** | Very rare - write-heavy with overwrites | Creates huge SSTables, avoid |

**STCS should almost never be used on modern systems.** It creates unmanageably large SSTables, results in poor streaming performance, and inefficient space reclamation.  It also requires a significant amount of free space, 50% in the worst case.

### Pre-3.0 with Spinning Disks

On very old Cassandra versions (pre-3.0) with spinning disks, STCS may be the only viable option due to LCS write amplification on slow storage.

**However:** If you're running pre-3.0 Cassandra on spinning disks, you should prioritize upgrading to modern Cassandra on SSDs. The performance, operational, and cost benefits far outweigh migration effort.

## Migration to UCS

### From STCS

`T4` provides 4 SSTables per tier, emulating STCS behavior while avoiding massive SSTables.

```cql
ALTER TABLE mykeyspace.foo WITH COMPACTION = {
    'class': 'UnifiedCompactionStrategy',
    'scaling_parameters': 'T4'
};
```


### From LCS

`L10` provides 10 SSTables per level, maintaining LCS-like read performance.

```cql
ALTER TABLE mykeyspace.foo WITH COMPACTION = {
    'class': 'UnifiedCompactionStrategy',
    'scaling_parameters': 'L10'
};
```

### From TWCS

The tiered settings can be used to reduce write amplification at higher levels.  
This provides similar behavior to TWCS without the negative effects of major compaction as the window is closed, 
which negatively impacts streaming performance.  Node density can be significantly increased by switching from TWCS to UCS.

```cql
ALTER TABLE mykeyspace.foo WITH COMPACTION = {
    'class': 'UnifiedCompactionStrategy',
    'scaling_parameters': 'T4,T8',
    'base_shard_count': '8'
};
```

## Scaling Parameters

| Parameter | Behavior | Use Case |
|-----------|----------|----------|
| `T4` | 4 SSTables per tier (tiered) | Write-heavy, STCS replacement |
| `T4,T8` | Tiered first level, fan out second | High-write time series |
| `L10` | 10 SSTables per level (leveled) | Read-heavy, LCS replacement |

## Throughput Tuning

### Key Settings
```yaml
compaction_throughput_mb_per_sec: 64  # Adjust based on workload
```

### Cassandra 5.0.4+ Optimization

- CASSANDRA-15452 provides 2-3x throughput improvement
- 3x IOPS reduction on cloud storage (EBS)
- Uses 256KB sequential reads for better cloud storage efficiency

## Monitoring

### Key Metrics
1. **Pending compactions**: Should be near zero normally
2. **SSTables per read**: Track via `nodetool tablehistograms`
3. **Compaction throughput**: Completion speed
4. **p99 read latency**: Ensure SLO compliance

### Commands
```bash
nodetool compactionstats
nodetool compactionhistory
nodetool tablehistograms keyspace.table
```

## Disk Space Requirements

- **Minimum**: 50% free space for worst-case compaction with STCS.
- **Critical**: You need at least enough space to perform a compaction. The old advisery of 50% does not apply to UCS and LCS.  I have seen clusters run at > 90% disk capacity, although some operations become high risk, such as snapshots.  If compaction falls behind or there's an influx of data, it can cause compaction to stop, resulting in full disks and write failures.  Keeping at least 100GB free is my minimum recommendation.
- Monitor continuously with alerting

## Common Issues

### High Pending Compactions

- Increase `compaction_throughput_mb_per_sec` if CPU and IO is not saturated.
- Upgrade to 5.0.4+ for better throughput on disaggregated storage such as EBS.
- If CPU or disk is saturated, you likely need to expand.

### Read Performance Degradation

- High SSTables per read indicates compaction behind or incorrect strategy / options are set.
- Switch to UCS with leveling parameters (L10) for read heavy or latency heavy workloads. 
- Review data model for large partitions, use `nodetool tablehistograms` to identify p99 and largest partitions.

## Summary

| Version | Recommendation                           |
|---------|------------------------------------------|
| 5.0+ | UCS for all workloads                    |
| 4.x/3.x | LCS for general, TWCS for time series.   |
| Pre-3.0 + spinning disks | STCS (but upgrade ASAP)                  |
| Any | Maintain 50% free disk                   |
