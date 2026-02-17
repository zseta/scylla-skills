# Cassandra Expert

A Claude Code plugin providing expert guidance for Apache Cassandra development and operations.

## Skills

This plugin includes four specialized skills:

| Skill | Purpose |
|-------|---------|
| `/cassandra-expert:diagnose` | Systematic troubleshooting using USE method, outlier analysis, double loop learning |
| `/cassandra-expert:optimize` | Performance tuning, configuration analysis, JVM settings, compaction strategies |
| `/cassandra-expert:data-model` | Schema design, partition keys, time-series modeling, query patterns |
| `/cassandra-expert:expert` | General Cassandra questions, CQL analysis, best practices |

## Installation

### From GitHub

```
/plugin marketplace add rustyrazorblade/cassandra-expert
/plugin install cassandra-expert@rustyrazorblade-plugins
```

### Local Development

```
/plugin marketplace add ./path/to/cassandra-expert
/plugin install cassandra-expert@rustyrazorblade-plugins
```

## Usage

### Diagnose - Troubleshooting Issues

```
/cassandra-expert:diagnose Read latencies spiking on some nodes but not others
/cassandra-expert:diagnose Compaction can't keep up with writes
/cassandra-expert:diagnose Getting timeouts during repair
```

### Optimize - Performance Tuning

```
/cassandra-expert:optimize Review my cassandra.yaml settings
/cassandra-expert:optimize What compaction strategy should I use?
/cassandra-expert:optimize Help tune GC for high-throughput workload
```

### Data Model - Schema Design

```
/cassandra-expert:data-model Design schema for user activity tracking
/cassandra-expert:data-model How should I model time-series data with 90-day retention?
/cassandra-expert:data-model Review this schema for hot partition risks
```

### Expert - General Questions

```
/cassandra-expert:expert What consistency level should I use for multi-DC?
/cassandra-expert:expert Review this CQL query for anti-patterns
/cassandra-expert:expert When should I use lightweight transactions?
```

## Problem-Solving Methodology

### Double Loop Learning
Goes beyond immediate fixes to identify root causes and prevent recurrence.

### USE Method
Systematically analyzes Utilization, Saturation, and Errors for CPU, memory, disk, network, storage, and thread pools.

### Outlier Analysis
Compares nodes to identify anomalies in latency, resource usage, and behavior.

### Configuration Analysis
Reviews system settings, cassandra.yaml, JVM settings, and per-table configurations.

## Key Recommendations

This plugin includes opinionated guidance based on real-world experience:

- **num_tokens**: Use 1 or max 4. Higher is always worse.
- **Compaction**: UCS for Cassandra 5.0+. Never use STCS.
- **Read-ahead**: Disable it. One of the worst settings you can have enabled.
- **Partitions**: Keep under 10MB. No downside to smaller partitions.
- **Row cache**: Keep disabled. Rarely beneficial.

## Requirements

- Claude Code 1.0.33 or later

## License

Apache 2.0
