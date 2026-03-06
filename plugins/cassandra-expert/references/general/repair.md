# Repair in Cassandra

## Overview

Repair ensures data consistency across replicas. When nodes miss writes due to failures or network partitions, repair identifies and resolves inconsistencies. Without regular repair, deleted data can resurrect and inconsistencies accumulate.

## Repair Methods

### Incremental Repair (4.0+ Only)

Tracks which SSTables have been repaired. Subsequent repairs only validate new/modified data.

```bash
nodetool repair
```

**Benefits:**
- Dramatically reduced I/O (only repairs new data since last run)
- Predictable resource usage proportional to write rate
- Faster completion (hours vs days for dense nodes)

Incremental repair can be used with subrange repair.  

**Warning:** Repairing a small range can result in significant anti compaction.  

**Critical:** Only safe on Cassandra 4.0+. The 4.0 release completely rewrote incremental repair.  
Before 4.0 there is a significant risk of repaired SSTables getting out of sync resulting in a massive flood of small SSTables
streaming to multiple nodes concurrently.  This poses a significant risk to availability.


### Full Repair

Repairs all data by building merkle trees for entire dataset.

```bash
nodetool repair --full
```

**Characteristics:**
- Cost scales linearly with data volume
- 10TB node may take 72+ hours
- High I/O and CPU impact

### Subrange Repair

Repairs specific token ranges. Required for pre-4.0 clusters.

```bash
nodetool repair -st <start_token> -et <end_token>
```

**Use cases:**
- Pre-4.0 clusters (mandatory)
- Targeted repair of specific ranges
- Managed by tools like Reaper

## Version-Specific Guidance

### Cassandra 4.0+

- **Use regular incremental repair** - it's safe and efficient
- Run regularly (within gc_grace_seconds)
- Monitor completion and failures

### Cassandra 3.x and Earlier

**DO NOT USE incremental repair.** Known issues include:

- **CASSANDRA-9143**: SSTable compaction race causes inconsistent repaired state
- **LCS SSTable proliferation**: Can crash nodes with thousands of L0 SSTables
- **Anti-compaction overhead**: First run can take days on large datasets
- **Concurrent repair prohibition**: Cannot run on multiple nodes simultaneously
- **Overstreaming**: Merkle tree leaf comparison streams unnecessary data

**For pre-4.0:** Use subrange repair exclusively via Reaper or similar tools.

## Relationship to gc_grace_seconds

Repair must complete within `gc_grace_seconds` to prevent data resurrection:

```
gc_grace_seconds >= (max_node_outage + repair_interval)
```

- Default gc_grace: 10 days (864000 seconds)
- Repair should complete in less than gc_grace
- If repair can't complete in time, increase gc_grace or fix repair

## Impact of vnodes

More vnodes = slower repair:
- Each token range requires separate merkle tree
- 256 vnodes creates massive repair overhead
- Use 1-4 vnodes for optimal repair performance

See: `vnodes.md`

## STCS and Subrange Repair Warning

When using incremental repair with subranges:

- Cassandra must split SSTables at token boundaries (anti-compaction)
- STCS creates massive SSTables (100s of GB), resulting in very long anti-compaction times.
- Splitting these repeatedly causes repair to balloon by orders of magnitude
- Nodes spend all time rewriting data, not repairing
- Counter-intuitively, using smaller ranges results in more anti-compaction, and worse performance.

**Solution:** Use UCS (5.0+) or LCS to limit SSTable sizes.

## Operational Best Practices

### Scheduling

- Run repair continuously or on regular schedule
- Complete full cluster repair within gc_grace_seconds
- Use tools like Reaper or AxonOps for automation

### Monitoring

```bash
# Check repair status
nodetool repair_admin list

# View repair history
nodetool repair_admin summarize
```

Track:
- Repair completion rate
- Time to complete full cluster repair
- Repair failures and retries
- Resource usage during repair

### During Repair

- Expect increased latency
- Monitor for dropped messages
- Watch disk space (streaming uses temporary space)
- Check for repair failures in logs

## Tools

### Cassandra Reaper

- Automated repair scheduling
- Subrange repair management
- Web UI for monitoring
- Handles failures and retries

### AxonOps
- Managed repair service
- Integrated monitoring
- Automatic scheduling

### Internal Repair Solution (6.0+)

CEP-37 (CASSANDRA-19918) incorporates repair into the Cassandra process, removing the need for external repair systems.


## Common Issues

### Repair Never Completes

- Too much data for repair window
- Use incremental repair (4.0+)
- Reduce vnode count
- Increase parallelism carefully

### Repair Failures
- Check logs for root cause
- Verify network connectivity
- Check disk space on all nodes
- May need to restart and resume

### Data Resurrection

- Repair didn't complete within gc_grace
- Tombstones purged before all replicas saw them
- Increase gc_grace or fix repair schedule

## Summary

- Always repair within gc_grace_seconds 
- Never use incremental repair on pre-4.0
- Automate with Reaper or similar
- Monitor completion and failures
