---
tags:
  - clickhouse
  - personal-documentation
  - analytical-database
  - reference
---

## My ClickHouse Knowledge Base

This is my comprehensive ClickHouse reference guide, organized for quick lookup and deep understanding. Each section builds on previous concepts while standing alone as a reference.

## Core Architecture & Fundamentals

### [[01-Clickhouse Fundamentals]]
- OLTP vs OLAP comparison
- Why columnar storage matters
- When to use ClickHouse vs PostgreSQL
- Core architectural decisions

### [[02-Vectorized Processing Pipeline]]
- How vectorized execution works
- SIMD processing explained
- Block-based processing model
- Query execution flow

### [[03-Storage Engine Architecture]]
- Physical data organization
- Granules and parts structure
- Sparse indexing system
- Background merge processes

## Compression & Storage

### [[04-Compression Deep Dive]]
- LZ4 vs ZSTD comparison
- Delta, T64, Gorilla codecs
- Two-phase compression strategy
- Real performance numbers

### [[05-MergeTree Engines Guide]]
- Base MergeTree engine
- ReplacingMergeTree for upserts
- SummingMergeTree for metrics
- CollapsingMergeTree for updates
- AggregatingMergeTree for complex aggregations

## Distributed Systems & Memory

### [[06-Distributed Architecture]]
- Sharding strategies
- Replication setup
- Distributed query execution
- Two-phase aggregation

### [[07-Memory Management]]
- Arena allocation pattern
- Disk spillover mechanisms
- Large operation handling
- Cache optimization

## SQL & Development

### [[08-ClickHouse SQL Reference]]
- CRUD operations differences
- Advanced aggregation functions
- Window functions and analytics
- Data type optimizations

### [[09-Schema Design Patterns]]
- Table design best practices
- Primary key strategies
- Partitioning guidelines
- Materialized views
- Denormalization patterns

### [[10-ETL and Integration]]
- PostgreSQL migration
- Kafka streaming setup
- S3 integration
- Python client usage
- Data quality monitoring

## 🔧 Operations & Performance

### [[11-Performance-Optimization]] *(Coming Next)*
- Query tuning techniques
- Cluster optimization
- Index utilization
- Compression tuning

## Quick Reference

### Most Important Concepts
1. **Columnar storage** reduces I/O by 10-20x
2. **Vectorized processing** uses SIMD for 4-8x speedup
3. **Background merges** optimize storage continuously
4. **Eventual consistency** trades ACID for performance
5. **Distributed aggregation** uses two-phase pattern

### Key Settings I Use
```sql
-- Query optimization
SET max_block_size = 65536;
SET max_threads = 16;
SET max_memory_usage = 10000000000;

-- Aggregation tuning
SET group_by_two_level_threshold = 100000;
SET max_bytes_before_external_group_by = 20000000000;
```

### Common Query Patterns
```sql
-- Time-series analysis
SELECT 
    toStartOfMonth(timestamp) as month,
    avg(value) as avg_value
FROM metrics 
WHERE timestamp >= '2024-01-01'
GROUP BY month
ORDER BY month;

-- Funnel analysis  
SELECT 
    step,
    uniq(user_id) as users,
    users / lag(users) OVER (ORDER BY step) as conversion_rate
FROM user_events
GROUP BY step
ORDER BY step;
```

## 📚 Learning Path

### Beginner (Start Here)
1. [[01-Clickhouse Fundamentals]] - Core concepts and OLAP vs OLTP
2. [[02-Vectorized Processing Pipeline]] - How queries work at the engine level
3. [[08-ClickHouse SQL Reference]] - SQL differences and analytical functions
4. [[09-Schema Design Patterns]] - Design basics and optimization

### Intermediate 
1. [[03-Storage Engine Architecture]] - Storage internals and physical layout
2. [[04-Compression Deep Dive]] - Performance optimization through compression
3. [[05-MergeTree Engines Guide]] - Engine selection for different patterns
4. [[10-ETL and Integration]] - Data pipeline setup and migration strategies

### Advanced
1. [[06-Distributed Architecture]] - Scaling concepts and cluster management
2. [[07-Memory Management]] - Memory optimization and large-scale processing
3. [[11-Performance-Optimization]] - Advanced tuning and troubleshooting
4. [[12-Troubleshooting-Guide]] - Production problem solving


## Key Technical Insights Covered

- **Vectorized Processing**: How ClickHouse achieves 10-100x performance improvements through SIMD
- **Compression Magic**: Achieving 10-30x storage reduction through specialized codecs and algorithms  
- **Distributed Query Execution**: Two-phase aggregation patterns across cluster nodes
- **MergeTree Engine Variants**: Choosing the right engine for different data update patterns
- **Memory Management**: Arena allocation and automatic disk spillover for large operations
- **Schema Design**: Denormalization strategies and primary key optimization for analytical workloads

## Practical Implementation Examples

Each guide contains:
- **Real configuration examples** from production deployments
- **Performance monitoring queries** for operational excellence
- **Common pitfall avoidance** with specific solutions
- **Best practices checklists** for implementation validation
- **Troubleshooting guides** with actual debugging techniques

---

*This documentation captures everything from my ClickHouse learning journey. Each file contains practical examples, performance insights, and real-world lessons learned from implementing analytical systems at scale.*