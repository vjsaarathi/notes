---
tags:
  - clickhouse-basics
  - olap-vs-oltp
  - columnar-database
  - architecture-overview
---

## What is ClickHouse?

ClickHouse = **Columnar analytical database** built for **OLAP workloads**

Key insight: It's designed for **reading massive amounts of data fast**, not for transactional operations like PostgreSQL.

## OLTP vs OLAP - The Core Difference

### OLTP (PostgreSQL, MySQL)
- **Purpose**: Handle individual transactions
- **Queries**: `SELECT * FROM orders WHERE order_id = 123`
- **Pattern**: Small, frequent operations
- **Storage**: Row-based (entire record stored together)
- **Consistency**: ACID transactions, immediate consistency

### OLAP (ClickHouse) 
- **Purpose**: Analyze large datasets
- **Queries**: `SELECT product_category, SUM(revenue) FROM orders GROUP BY product_category`
- **Pattern**: Large scans with aggregations
- **Storage**: Column-based (each column stored separately)
- **Consistency**: Eventually consistent, optimized for reads

**Mental model**: PostgreSQL = Cash register, ClickHouse = Data warehouse

## Columnar Storage - The Game Changer

### Row Storage (Traditional)
```
Row 1: [id=123, customer="John", price=50.00, date="2024-01-15"]
Row 2: [id=124, customer="Jane", price=75.00, date="2024-01-16"]
```

**Problem**: To calculate `SUM(price)`, I have to read customer names, IDs, dates - everything!

### Columnar Storage (ClickHouse)
```
ID column:       [123, 124, 125, ...]
Customer column: ["John", "Jane", "Bob", ...]
Price column:    [50.00, 75.00, 32.50, ...]
Date column:     ["2024-01-15", "2024-01-16", ...]
```

**Advantage**: For `SUM(price)`, only read the price column = **10-20x less I/O**

### Why This Matters for Analytics

**Example query**: "What's our daily revenue for the last year?"

- **Row storage**: Read 100GB to scan all columns
- **Columnar**: Read 5GB (only price + date columns)
- **Result**: 20x faster query execution

## Compression Magic

Columnar data compresses incredibly well because similar values are stored together:

### Status Column Example
```
Row storage:    "shipped", "pending", "shipped", "delivered", "shipped"...
Columnar:       ["shipped", "shipped", "shipped", "pending", "delivered"]
                 ↓ (groups similar values)
Compressed:     "shipped" x 1000, "pending" x 200, "delivered" x 150
```

**Real numbers I've seen**: 10-30x compression ratios are common

**Combined benefit**: 
- 20x less I/O (columnar)
- 10x less storage (compression)
- = **200x performance improvement potential**

## When to Use ClickHouse

### Perfect Use Cases

**Time-series analytics**
- IoT sensor data analysis
- Application metrics and monitoring
- Financial market data

**Business intelligence**
- Revenue dashboards
- Customer behavior analysis
- Product performance reports

**Event data processing**
- Web analytics (clickstream)
- A/B testing analysis
- Log aggregation

### Bad Fit

**Transactional systems**
- Order processing
- User account management
- Real-time inventory

**Frequent updates**
- Social media posts (constant editing)
- Real-time gaming leaderboards
- Chat applications

**Small datasets**
- < 1 million records
- Simple CRUD apps

## Key Architectural Decisions

### why ClickHouse behaves differently:

### 1. Read-Optimized Over Write-Optimized
- Complex background processes for optimal reads
- Writes may have higher latency
- **Trade-off**: Amazing query performance, more complex data ingestion

### 2. Immutable Data Model
- Data parts are never modified in-place
- Updates create new data parts
- **Trade-off**: Simple concurrency model, but no traditional UPDATE/DELETE

### 3. Eventual Consistency
- Background merges optimize data layout
- Query results are eventually consistent
- **Trade-off**: Higher performance, but application complexity

### 4. Denormalization-Friendly
- Pre-join data for better performance
- Materialized views for pre-aggregation
- **Trade-off**: Storage overhead for query speed

## Performance Characteristics

Numbers that convinced me to learn ClickHouse:

| Workload | ClickHouse vs PostgreSQL | Why |
|----------|-------------------------|-----|
| **Time-series aggregation** | 50-200x faster | Columnar + compression |
| **Large table scans** | 10-100x faster | I/O reduction |
| **Complex analytics** | 20-50x faster | Vectorized processing |
| **Dashboard queries** | 5-25x faster | Pre-aggregation |

**Real example from my experience**:
- Query: Daily revenue for 2 years of e-commerce data
- PostgreSQL: 45 seconds
- ClickHouse: 0.8 seconds
- **56x improvement**

## Mental Models That Help Me

### 1. The Library Analogy
- **Columnar storage** = Books organized by subject (all history books together)
- **Row storage** = Books organized by acquisition date (random subjects mixed)
- **Query** = "Find all books about WWII"
- **Columnar wins** = Only search history section
- **Row storage** = Search entire library

### 2. The Data Pipeline Analogy
- **OLTP** = Assembly line (one item at a time)
- **OLAP** = Batch processing factory (thousands at once)
- **Vectorization** = Multiple machines working in parallel

### 3. The Warehouse Analogy
- **Background merges** = Warehouse reorganization during off-hours
- **Parts** = Pallets of goods
- **Compression** = Efficient packing
- **Granules** = Individual boxes within pallets

## Key Mindset Shifts from PostgreSQL

1. **Think in batches**: Insert thousands of rows, not individual records
2. **Design for immutability**: Append data, don't update
3. **Embrace denormalization**: Pre-join for performance
4. **Plan for eventually consistent reads**: Design application accordingly
5. **Optimize for analytical patterns**: Structure for aggregations, not lookups

## First Questions I Ask for Any Use Case

1. **Read vs Write ratio**: >90% reads = good fit for ClickHouse
2. **Query patterns**: Aggregations and analytics = yes, lookups = no
3. **Data volume**: >1M records and growing = worth considering
4. **Update frequency**: Rare updates = perfect, frequent = reconsider
5. **Consistency requirements**: Eventual consistency OK = good fit

## Common Misconceptions I Had

### "ClickHouse can't handle updates"
**Reality**: It can, but through different patterns (ReplacingMergeTree, mutations)

### "It's only for massive datasets"
**Reality**: Benefits start showing at ~1M records, massive improvements at 10M+

### "SQL compatibility issues"
**Reality**: 95% compatible with standard SQL, main differences in UPDATE/DELETE patterns

---

**Next**: [[02-Vectorized Processing Pipeline]] - How ClickHouse processes queries so fast
