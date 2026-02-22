# Cassandra 4.0 Notable Features

## Zero-Copy Streaming

Zero-Copy Streaming transfers entire SSTables between nodes without reifying them into objects. It uses the Linux `sendfile()` system call to transfer data directly from filesystem to network.

### Benefits
- **5x faster transfers** compared to partition-based streaming
- **Reduced GC pressure** on sending and receiving nodes
- **Faster node replacement** and cluster recovery
- Throughput limited only by disk and network I/O

### Requirements
- Enabled by default: `stream_entire_sstables: true`
- Only applies when all partitions within an SSTable are being transferred
- **Disabled automatically with internode encryption enabled**

### Maximizing ZCS Eligibility

Zero-Copy Streaming only works when the destination node owns the complete token range of an SSTable. To maximize eligibility:

- Use UCS or LCS (bounded SSTable sizes)
- Limit num_tokens to 1-4 (see `../general/vnodes.md`)
- Avoid STCS (creates unbounded SSTable sizes)

With bounded SSTable sizes, new nodes are more likely to own complete SSTable token ranges, meaning more SSTables qualify for the fast Zero-Copy path.

Reference: [Streaming and Compaction](https://rustyrazorblade.com/post/2025/03-streaming/)

## Virtual Tables

Cassandra 4.0 introduced virtual tables for observability:

- Query system state via CQL
- No data stored on disk
- Examples: `system_views.sstable_tasks`, `system_views.thread_pools`

## Audit Logging

Full query logging for compliance and debugging:

- Log all CQL queries
- Configurable filters
- Multiple output formats

## Full Query Logging (FQL)

Detailed query capture for replay and analysis:

- Capture queries with timing information
- Replay against test clusters
- Performance analysis

## Incremental Repair Improvements

More reliable incremental repair:

- Better consistency
- Reduced overhead
- Improved anti-compaction
