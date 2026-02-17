# Cassandra Expert

A Claude Code plugin providing expert guidance for Apache Cassandra development and operations.

## Features

- **Data Modeling** - Query-first schema design, partition key selection, denormalization strategies
- **CQL Query Analysis** - Identify anti-patterns, optimize queries, proper index usage
- **Performance Tuning** - Compaction strategies, consistency levels, JVM tuning
- **Troubleshooting** - Systematic problem-solving using USE method, outlier analysis, config review
- **Cluster Operations** - Replication, repairs, migrations, upgrades

## Installation

### From GitHub (Marketplace)

```
/plugin marketplace add rustyrazorblade/cassandra-expert
/plugin install cassandra-expert
```

### Local Development

```bash
claude --plugin-dir /path/to/cassandra-expert
```

## Usage

Invoke the skill directly:

```
/cassandra-expert:cassandra-expert How should I model time-series data with a 90-day retention?
```

Or ask Cassandra-related questions and Claude will automatically use this skill when appropriate.

### Example Questions

- "Review this CQL schema for a user activity tracking system"
- "Why are my read latencies spiking on some nodes?"
- "What compaction strategy should I use for time-series data?"
- "Help me diagnose why repairs are taking too long"
- "Is this query pattern going to cause hot partitions?"

## Problem-Solving Methodology

This plugin applies structured approaches to Cassandra troubleshooting:

### Double Loop Learning
Goes beyond immediate fixes to identify root causes and prevent recurrence.

### USE Method
Systematically analyzes Utilization, Saturation, and Errors for CPU, memory, disk, network, and storage.

### Outlier Analysis
Compares nodes to identify anomalies in latency, resource usage, and behavior.

### Configuration Analysis
Reviews cassandra.yaml, JVM settings, and per-table configurations for issues.

## Requirements

- Claude Code 1.0.33 or later

## License

Apache 2.0
