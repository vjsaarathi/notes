---
tags:
  - clickhouse-sql
  - sql-dialect-differences
  - analytical-functions
  - aggregation-functions
  - window-functions
  - data-types
---

## SQL Dialect Philosophy

ClickHouse SQL is ~95% compatible with standard SQL but optimized for analytical workloads. The key differences I learned:

**Traditional SQL focus**: ACID transactions, row-level operations, normalization
**ClickHouse SQL focus**: Bulk operations, columnar processing, analytical functions, eventual consistency

**Mental shift**: Think in terms of **data analysis** rather than **data management**.

## CRUD Operations - The Big Differences

### INSERT Operations

**Traditional SQL (PostgreSQL)**:
```sql
-- Row-by-row inserts (typical OLTP pattern)
INSERT INTO orders VALUES (1, 100, '2024-01-15', 50.00);
INSERT INTO orders VALUES (2, 101, '2024-01-15', 75.00);  -- Each creates transaction
```

**ClickHouse approach**:
```sql
-- Batch inserts (analytical pattern)
INSERT INTO orders VALUES 
    (1, 100, '2024-01-15', 50.00),
    (2, 101, '2024-01-15', 75.00),
    (3, 102, '2024-01-15', 32.50),
    -- ... thousands of rows in single batch
    (10000, 199, '2024-01-15', 89.99);

-- From other tables (common pattern)
INSERT INTO orders SELECT * FROM staging_orders WHERE processed = 0;

-- From external sources
INSERT INTO orders SELECT * FROM file('data.csv', 'CSV');
INSERT INTO orders SELECT * FROM s3('bucket/path/*.parquet');
```

### UPDATE Operations - Engine Dependent

**Standard MergeTree** (No traditional UPDATE):
```sql
-- This doesn't exist in base MergeTree
-- UPDATE orders SET status = 'shipped' WHERE order_id = 123;  -- ERROR!

-- Instead: Use mutations (heavy operations)
ALTER TABLE orders UPDATE status = 'shipped' WHERE order_id = 123;

-- Or: Insert new data with updated values
INSERT INTO orders SELECT order_id, customer_id, order_date, total_amount, 'shipped' as status
FROM orders WHERE order_id = 123;
```

**ReplacingMergeTree** (Implicit updates):
```sql
-- Original record
INSERT INTO user_profiles VALUES (123, 'John Doe', 'john@email.com', 1);

-- "Update" by inserting new version (higher version number)
INSERT INTO user_profiles VALUES (123, 'John Smith', 'john.smith@email.com', 2);

-- Background merge keeps only highest version
-- Query for current state:
SELECT * FROM user_profiles FINAL WHERE user_id = 123;  -- Gets version 2
-- Or handle in application:
SELECT * FROM user_profiles WHERE user_id = 123 ORDER BY version DESC LIMIT 1;
```

**CollapsingMergeTree** (Explicit cancel/insert pattern):
```sql
-- Original record  
INSERT INTO user_sessions VALUES (123, 'session_456', '10:00', '10:30', 5, 1);  -- sign=1

-- Update by canceling + inserting new
INSERT INTO user_sessions VALUES 
    (123, 'session_456', '10:00', '10:30', 5, -1),   -- Cancel original (sign=-1)
    (123, 'session_456', '10:00', '11:00', 8, 1);    -- New version (sign=1)

-- Always filter by sign for current state
SELECT * FROM user_sessions WHERE sign = 1 AND user_id = 123;
```

### DELETE Operations

**Lightweight Deletes** (22.8+, Experimental):
```sql
-- Modern ClickHouse approach
SET allow_experimental_lightweight_delete = 1;
DELETE FROM orders WHERE order_date < '2020-01-01';

-- Creates deletion mask, data eventually removed during merges
-- Immediate effect on queries, but data persists on disk temporarily
```

**Mutation-based Deletes**:
```sql
-- Traditional heavy approach - rewrites affected parts
ALTER TABLE orders DELETE WHERE order_date < '2020-01-01';

-- Monitor mutation progress
SELECT command, is_done FROM system.mutations WHERE table = 'orders';
```

**Partition Drops** (Most efficient for large deletions):
```sql
-- For time-partitioned tables, extremely fast
ALTER TABLE orders DROP PARTITION '202012';  -- Drops entire December 2020 partition
ALTER TABLE orders DROP PARTITION '202011';  -- Much faster than DELETE WHERE
```

## Data Types - Optimized for Analytics

### Numeric Types with Performance Focus

```sql
-- Integer types optimized for compression and vectorization
CREATE TABLE metrics (
    -- Choose smallest type that fits your range
    device_id UInt16,              -- 0 to 65,535 (2 bytes)
    user_id UInt32,                -- 0 to 4.3 billion (4 bytes)  
    timestamp UInt32,              -- Unix timestamp (4 bytes vs 8 for DateTime)
    
    -- Decimal for exact arithmetic
    price Decimal(10,2),           -- Up to 99,999,999.99
    
    -- Float for approximate calculations (better compression)
    sensor_value Float32,          -- Single precision for sensors
    calculation_result Float64     -- Double precision for aggregations
) ENGINE = MergeTree() ORDER BY (timestamp, device_id);
```

### String Types and Encoding

```sql
CREATE TABLE user_data (
    -- Fixed strings for better compression when length is predictable
    country_code FixedString(2),           -- 'US', 'EU' etc (always 2 chars)
    postal_code FixedString(10),           -- Padded to 10 chars
    
    -- Variable strings for unpredictable lengths  
    user_name String,                      -- Variable length
    description String,
    
    -- Low cardinality for categorical data
    status LowCardinality(String),         -- Few distinct values, great compression
    category LowCardinality(String),
    
    -- Enums for known fixed sets
    user_type Enum8('free'=1, 'premium'=2, 'enterprise'=3)
) ENGINE = MergeTree() ORDER BY user_id;
```

### Date and Time Types

```sql
CREATE TABLE events (
    -- Choose precision based on needs
    event_date Date,               -- Day precision, 2 bytes
    event_datetime DateTime,       -- Second precision, 4 bytes  
    event_datetime64 DateTime64(3), -- Millisecond precision, 8 bytes
    
    -- Timezone handling
    created_at DateTime('UTC'),           -- Store in UTC
    user_local_time DateTime('America/New_York'),  -- User's timezone
    
    -- For analytics, often Date is sufficient
    partition_date Date MATERIALIZED toDate(event_datetime),  -- Derived column
    hour_of_day UInt8 MATERIALIZED toHour(event_datetime)     -- For time-of-day analysis
) ENGINE = MergeTree() 
ORDER BY (event_date, event_datetime)
PARTITION BY toYYYYMM(event_date);
```

### Array and Complex Types

```sql
CREATE TABLE user_behavior (
    user_id UInt32,
    
    -- Arrays for multi-value attributes
    visited_pages Array(String),
    purchase_amounts Array(Decimal(10,2)),
    event_timestamps Array(DateTime),
    
    -- Nested types for structured data
    sessions Nested(
        session_id String,
        start_time DateTime, 
        duration_minutes UInt32,
        page_count UInt16
    ),
    
    -- JSON for semi-structured data (22.8+)
    user_properties JSON,
    
    -- Maps for key-value pairs  
    custom_attributes Map(String, String)
) ENGINE = MergeTree() ORDER BY user_id;

-- Query arrays and nested data
SELECT 
    user_id,
    length(visited_pages) as page_count,
    arrayElement(visited_pages, 1) as first_page,
    sessions.session_id as all_session_ids,
    sessions.duration_minutes as all_durations
FROM user_behavior
WHERE has(visited_pages, '/checkout');  -- Array membership test
```

## Advanced Aggregation Functions

### Standard vs ClickHouse Aggregations

**Basic aggregations** (same as standard SQL):
```sql
SELECT 
    product_category,
    COUNT(*) as order_count,
    SUM(total_amount) as revenue,
    AVG(total_amount) as avg_order_value,
    MIN(order_date) as first_order,
    MAX(order_date) as last_order
FROM orders
GROUP BY product_category;
```

**ClickHouse-specific aggregations** (analytical powerhouses):
```sql
SELECT 
    product_category,
    
    -- Distinct counting (multiple approaches)
    COUNT(DISTINCT customer_id) as exact_customers,     -- Exact but slower
    uniq(customer_id) as approx_customers,              -- ~1% error, much faster
    uniqExact(customer_id) as exact_customers_v2,       -- Same as COUNT DISTINCT
    uniqHLL12(customer_id) as hll_customers,            -- HyperLogLog with 12 bits
    
    -- Quantile functions  
    quantile(0.5)(total_amount) as median_order,        -- Approximate median
    quantileExact(0.5)(total_amount) as exact_median,   -- Exact median (slower)
    quantiles(0.25, 0.5, 0.75)(total_amount) as quartiles,  -- Multiple quantiles
    
    -- Statistical functions
    stddevPop(total_amount) as amount_stddev,
    corr(total_amount, quantity) as amount_qty_correlation,
    
    -- Array aggregations
    groupArray(customer_id) as all_customers,           -- Collect into array
    groupUniqArray(customer_id) as unique_customers,    -- Unique values as array
    arraySort(groupArray(order_date)) as sorted_dates,
    
    -- Conditional aggregations  
    countIf(total_amount > 100) as large_orders,
    sumIf(total_amount, total_amount > 100) as large_orders_revenue,
    avgIf(total_amount, customer_type = 'premium') as premium_avg
    
FROM orders
GROUP BY product_category;
```

### Time-Series Specific Functions

```sql
-- Time-series aggregations
SELECT 
    toStartOfHour(timestamp) as hour,
    
    -- Rate calculations
    count() / 3600 as events_per_second,               -- Event rate
    
    -- Moving averages (over arrays)
    arrayMap(x -> x/60, 
        arrayDifference(
            arraySort(groupArray(toUnixTimestamp(timestamp)))
        )
    ) as intervals_between_events,
    
    -- First/last value functions
    any(value) as any_value,                           -- Any value (fastest)
    anyLast(value) as last_value,                      -- Last inserted value
    anyHeavy(category) as most_frequent_category,      -- Heavy hitters algorithm
    
    -- Time-based windows
    lagInFrame(count(), 1) OVER (ORDER BY hour) as prev_hour_count,
    leadInFrame(count(), 1) OVER (ORDER BY hour) as next_hour_count
    
FROM events
GROUP BY hour
ORDER BY hour;
```

### State Functions for Pre-aggregation

These are crucial for **AggregatingMergeTree** and **materialized views**:

```sql
-- Create aggregation states (for AggregatingMergeTree)
SELECT 
    toDate(timestamp) as date,
    device_type,
    
    -- Create aggregation states (not final values!)
    countState() as event_count_state,              -- For count aggregation
    sumState(bytes_transferred) as bytes_state,      -- For sum aggregation  
    uniqState(user_id) as unique_users_state,       -- For distinct count
    quantileState(0.95)(response_time) as p95_state -- For quantile
    
FROM events
GROUP BY date, device_type;

-- Extract final values (for querying AggregatingMergeTree)
SELECT 
    date,
    device_type,
    countMerge(event_count_state) as total_events,
    sumMerge(bytes_state) as total_bytes,
    uniqMerge(unique_users_state) as unique_users,
    quantileMerge(0.95)(p95_state) as p95_response_time
FROM aggregated_events
GROUP BY date, device_type;
```

## Window Functions and Analytics

### Ranking and Row Numbers

```sql
SELECT 
    customer_id,
    order_date,
    total_amount,
    
    -- Ranking functions
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) as order_sequence,
    RANK() OVER (PARTITION BY customer_id ORDER BY total_amount DESC) as amount_rank,
    DENSE_RANK() OVER (ORDER BY total_amount DESC) as dense_amount_rank,
    
    -- Percentiles
    PERCENT_RANK() OVER (ORDER BY total_amount) as amount_percentile,
    CUME_DIST() OVER (ORDER BY total_amount) as cumulative_distribution,
    
    -- N-tiles
    NTILE(4) OVER (ORDER BY total_amount) as quartile,
    NTILE(100) OVER (ORDER BY total_amount) as percentile_bucket
    
FROM orders
ORDER BY customer_id, order_date;
```

### Lag/Lead and Time-Series Analysis

```sql
-- Customer behavior analysis
SELECT 
    customer_id,
    order_date,
    total_amount,
    
    -- Previous/next values
    LAG(total_amount, 1) OVER (PARTITION BY customer_id ORDER BY order_date) as prev_order_amount,
    LEAD(order_date, 1) OVER (PARTITION BY customer_id ORDER BY order_date) as next_order_date,
    
    -- Multiple lag values
    LAG(total_amount, 1, 0) OVER (PARTITION BY customer_id ORDER BY order_date) as prev_amount_default,
    LAG(total_amount, 2) OVER (PARTITION BY customer_id ORDER BY order_date) as two_orders_ago,
    
    -- Time between orders
    dateDiff('day', 
        LAG(order_date, 1) OVER (PARTITION BY customer_id ORDER BY order_date),
        order_date
    ) as days_since_last_order,
    
    -- Running calculations
    SUM(total_amount) OVER (PARTITION BY customer_id ORDER BY order_date) as running_total,
    AVG(total_amount) OVER (PARTITION BY customer_id ORDER BY order_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) as moving_avg_3_orders
    
FROM orders
ORDER BY customer_id, order_date;
```

### Advanced Window Functions

```sql
-- Cohort analysis and complex analytics
SELECT 
    customer_id,
    order_date,
    total_amount,
    
    -- First/last values in window
    FIRST_VALUE(total_amount) OVER (PARTITION BY customer_id ORDER BY order_date) as first_order_amount,
    LAST_VALUE(total_amount) OVER (PARTITION BY customer_id ORDER BY order_date ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) as last_order_amount,
    
    -- Frame-specific functions  
    COUNT(*) OVER (PARTITION BY customer_id ORDER BY order_date ROWS BETWEEN 30 PRECEDING AND CURRENT ROW) as orders_last_30,
    SUM(total_amount) OVER (PARTITION BY customer_id ORDER BY order_date RANGE BETWEEN INTERVAL 30 DAY PRECEDING AND CURRENT ROW) as revenue_last_30_days,
    
    -- Customer lifecycle analysis
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) as order_number,
    CASE 
        WHEN ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) = 1 THEN 'New'
        WHEN ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) = 2 THEN 'Repeat'
        ELSE 'Loyal'
    END as customer_segment
    
FROM orders
ORDER BY customer_id, order_date;
```

## Array Functions and Operations

### Array Creation and Manipulation

```sql
-- Working with arrays
SELECT 
    customer_id,
    
    -- Create arrays from aggregation
    groupArray(product_name) as purchased_products,
    groupArray(order_date) as order_dates,
    groupUniqArray(product_category) as unique_categories,
    
    -- Array operations
    arraySort(groupArray(total_amount)) as amounts_sorted,
    arrayReverse(arraySort(groupArray(order_date))) as dates_desc,
    arrayDistinct(groupArray(product_category)) as unique_cats_alt,
    
    -- Array element access
    arrayElement(groupArray(product_name), 1) as first_product,
    groupArray(product_name)[1] as first_product_alt,  -- Alternative syntax
    arraySlice(arraySort(groupArray(order_date)), 1, 3) as first_three_orders,
    
    -- Array statistics
    length(groupArray(product_name)) as total_products,
    arraySum(groupArray(total_amount)) as total_spent,
    arrayAvg(groupArray(total_amount)) as avg_order_amount
    
FROM orders
GROUP BY customer_id;
```

### Array Filtering and Searching

```sql
SELECT 
    customer_id,
    groupArray(product_name) as products,
    groupArray(product_category) as categories,
    groupArray(total_amount) as amounts,
    
    -- Array filtering
    arrayFilter(x -> x > 100, groupArray(total_amount)) as large_orders,
    arrayFilter((x, y) -> y = 'Electronics', 
                groupArray(product_name), 
                groupArray(product_category)) as electronics_products,
    
    -- Array searching
    has(groupArray(product_category), 'Electronics') as bought_electronics,
    indexOf(groupArray(product_category), 'Electronics') as first_electronics_position,
    arrayCount(x -> x > 50, groupArray(total_amount)) as orders_over_50,
    
    -- Array transformations  
    arrayMap(x -> x * 1.1, groupArray(total_amount)) as amounts_with_tax,
    arrayMap((name, cat) -> concat(cat, ': ', name), 
             groupArray(product_name), 
             groupArray(product_category)) as formatted_products
             
FROM orders  
GROUP BY customer_id;
```

## String Functions and Text Analytics

### String Manipulation

```sql
SELECT 
    -- Basic string functions
    concat('Customer: ', toString(customer_id)) as customer_label,
    upper(product_name) as product_upper,
    lower(product_category) as category_lower,
    length(description) as desc_length,
    
    -- String extraction
    substring(product_name, 1, 10) as short_name,
    extractAllGroups(email, '(.+)@(.+)\\.(com|org|net)') as email_parts,
    
    -- Pattern matching
    match(product_name, 'iPhone.*') as is_iphone,
    extract(phone_number, '\\d{3}-\\d{3}-\\d{4}') as formatted_phone,
    
    -- String splitting  
    splitByChar('-', product_code) as code_parts,
    splitByString(' | ', tags) as tag_array,
    arrayStringConcat(splitByChar(' ', product_name), '_') as snake_case_name,
    
    -- String replacement
    replaceAll(description, 'old_term', 'new_term') as updated_description,
    replaceRegexpAll(text, '\\d+', 'NUMBER') as anonymized_text
    
FROM products;
```

### Text Analytics Functions

```sql
-- Advanced text processing
SELECT 
    product_name,
    description,
    
    -- N-grams for text analysis
    ngramDistance(description, 'high quality product') as similarity_score,
    ngramSearch(description, 'premium quality') as contains_premium,
    
    -- Phonetic matching
    soundex(product_name) as phonetic_code,
    
    -- Text normalization
    normalizeQuery(search_query) as normalized_query,
    
    -- URL/domain extraction
    domain(website_url) as domain_name,
    topLevelDomain(website_url) as tld,
    
    -- Date parsing from strings
    parseDateTime64BestEffort(created_at_string) as parsed_timestamp,
    parseDateTimeBestEffort(date_string) as parsed_date
    
FROM products;
```

## Date and Time Functions

### Date Arithmetic and Extraction

```sql
SELECT 
    order_date,
    order_timestamp,
    
    -- Date extraction
    toYear(order_date) as order_year,
    toMonth(order_date) as order_month,  
    toDayOfWeek(order_date) as day_of_week,  -- 1=Monday, 7=Sunday
    toDayOfYear(order_date) as day_of_year,
    toHour(order_timestamp) as order_hour,
    
    -- Date rounding/truncation
    toStartOfMonth(order_date) as month_start,
    toStartOfWeek(order_date) as week_start,
    toStartOfDay(order_timestamp) as day_start,
    toStartOfHour(order_timestamp) as hour_start,
    toStartOfFifteenMinutes(order_timestamp) as fifteen_min_bucket,
    
    -- Date arithmetic
    addDays(order_date, 30) as thirty_days_later,
    subtractWeeks(order_date, 2) as two_weeks_ago,
    addMonths(order_date, 1) as next_month,
    
    -- Date differences
    dateDiff('day', order_date, today()) as days_since_order,
    dateDiff('hour', order_timestamp, now()) as hours_since_order,
    age('years', order_date, today()) as years_since_order
    
FROM orders;
```

### Time Zone Handling

```sql
-- Working with timezones
SELECT 
    event_timestamp,
    
    -- Convert between timezones
    toTimeZone(event_timestamp, 'America/New_York') as ny_time,
    toTimeZone(event_timestamp, 'Europe/London') as london_time,
    toTimeZone(event_timestamp, 'Asia/Tokyo') as tokyo_time,
    
    -- Extract timezone info
    toTimezone(event_timestamp) as current_timezone,
    
    -- Unix timestamp conversions
    toUnixTimestamp(event_timestamp) as unix_seconds,
    fromUnixTimestamp(unix_timestamp) as converted_datetime,
    
    -- Formatting dates
    formatDateTime(event_timestamp, '%Y-%m-%d %H:%i:%s') as formatted_date,
    toString(event_timestamp) as string_representation
    
FROM events;
```

## Conditional Logic and CASE Expressions

### Advanced CASE and Conditionals

```sql
SELECT 
    customer_id,
    total_amount,
    product_category,
    
    -- Multi-condition CASE
    CASE 
        WHEN total_amount >= 1000 THEN 'Premium'
        WHEN total_amount >= 500 THEN 'Standard'  
        WHEN total_amount >= 100 THEN 'Basic'
        ELSE 'Low Value'
    END as customer_tier,
    
    -- ClickHouse multiIf (more efficient than CASE)
    multiIf(
        total_amount >= 1000, 'Premium',
        total_amount >= 500, 'Standard',
        total_amount >= 100, 'Basic',
        'Low Value'
    ) as customer_tier_optimized,
    
    -- Conditional aggregations  
    IF(product_category = 'Electronics', total_amount, 0) as electronics_amount,
    IF(customer_type = 'premium', 1, 0) as is_premium,
    
    -- Null handling
    ifNull(discount_amount, 0) as discount_safe,
    nullIf(customer_notes, '') as notes_or_null,  -- Convert empty string to NULL
    assumeNotNull(customer_id) as customer_id_not_null,  -- Performance hint
    
    -- Coalescing  
    coalesce(preferred_name, first_name, 'Unknown') as display_name
    
FROM orders;
```

## Performance-Specific SQL Patterns

### Optimized Query Patterns

```sql
-- Efficient patterns I use regularly

-- 1. LIMIT BY for top-N per group (better than window functions)
SELECT customer_id, order_date, total_amount
FROM orders  
ORDER BY customer_id, total_amount DESC
LIMIT 5 BY customer_id;  -- Top 5 orders per customer

-- 2. SAMPLE for large dataset exploration  
SELECT avg(total_amount), count()
FROM orders SAMPLE 0.1  -- Process 10% sample
WHERE order_date >= '2024-01-01';

-- 3. PREWHERE for early filtering (before reading all columns)
SELECT customer_id, product_name, total_amount
FROM orders
PREWHERE order_date >= '2024-01-01'  -- Applied before reading product_name
WHERE total_amount > 100;            -- Applied after reading all columns

-- 4. Efficient EXISTS checks
SELECT DISTINCT customer_id
FROM orders o1
WHERE EXISTS (
    SELECT 1 FROM orders o2 
    WHERE o2.customer_id = o1.customer_id 
      AND o2.product_category = 'Electronics'
);

-- 5. Semi-joins with IN (often faster than EXISTS)
SELECT DISTINCT customer_id  
FROM orders
WHERE customer_id IN (
    SELECT customer_id FROM orders WHERE product_category = 'Electronics'
);
```

### Settings for SQL Query Optimization

```sql
-- Performance settings I use
SET max_threads = 16;                           -- Parallel processing
SET max_memory_usage = 10000000000;             -- 10GB memory limit
SET use_uncompressed_cache = 1;                 -- Enable caching
SET optimize_skip_unused_shards = 1;            -- Skip irrelevant shards
SET optimize_throw_if_noop = 1;                 -- Catch inefficient queries

-- Join optimizations
SET join_algorithm = 'hash';                    -- Hash joins
SET max_bytes_in_join = 1000000000;            -- 1GB join limit

-- GROUP BY optimizations  
SET group_by_two_level_threshold = 100000;      -- Two-level aggregation
SET max_bytes_before_external_group_by = 20000000000;  -- 20GB before spillover
```

## Common SQL Pitfalls and Solutions

### 1. Inefficient GROUP BY Patterns
```sql
-- Avoid: High cardinality GROUP BY
SELECT customer_id, order_id, SUM(amount)  -- Too many groups
FROM order_items 
GROUP BY customer_id, order_id;

-- Better: Hierarchical aggregation
SELECT customer_id, SUM(order_total)
FROM (
    SELECT customer_id, order_id, SUM(amount) as order_total
    FROM order_items
    GROUP BY customer_id, order_id  
) 
GROUP BY customer_id;
```

### 2. Wrong JOIN Order
```sql
-- Inefficient: Large table first
SELECT *
FROM large_orders lo
JOIN small_customers sc ON lo.customer_id = sc.customer_id;

-- Better: Small table first (ClickHouse builds hash table from first table)
SELECT *  
FROM small_customers sc
JOIN large_orders lo ON sc.customer_id = lo.customer_id;
```

### 3. Missing Primary Key Usage
```sql
-- Slow: Doesn't use primary key (order_date, customer_id)
SELECT * FROM orders WHERE customer_id = 12345;

-- Fast: Uses primary key prefix  
SELECT * FROM orders 
WHERE order_date >= '2024-01-01' AND customer_id = 12345;
```

### 4. Suboptimal String Operations
```sql
-- Slow: Function on every row
SELECT * FROM products WHERE lower(product_name) LIKE '%iphone%';

-- Faster: Use appropriate data types and indexes
SELECT * FROM products WHERE product_name_lower LIKE '%iphone%';
-- Or create LowCardinality column, or use full-text search
```

## Best Practices Summary

### Query Design Principles I Follow

1. **Filter early and often**: Use PREWHERE, optimize WHERE clauses
2. **Leverage primary key ordering**: Structure queries to use index efficiently  
3. **Batch operations**: Prefer bulk INSERT/SELECT over row-by-row operations
4. **Use appropriate functions**: Choose approximate vs exact based on needs
5. **Monitor query performance**: Use EXPLAIN and system.query_log analysis

### Function Selection Guidelines

- **Use `uniq()` over `COUNT(DISTINCT)`** for large datasets (faster, small error)
- **Use `quantile()` over `quantileExact()`** when approximation is acceptable  
- **Use `multiIf()` over `CASE`** for better vectorization
- **Use array functions** for complex multi-value analysis
- **Use window functions** for analytical calculations over ordered data

### Memory and Performance Considerations

- **LIMIT BY over window functions** for top-N queries
- **SAMPLE for exploration** of large datasets
- **Pre-aggregate with materialized views** for repeated calculations
- **Use appropriate data types** for compression and processing efficiency

**Key insight**: ClickHouse SQL is optimized for analytical thinking - focus on **what insights you want** rather than **how to manage transactions**. The dialect encourages bulk operations, approximation where acceptable, and vectorized processing patterns.

---

**Next**: [[09-Schema Design Patterns]] - Optimal table design and indexing strategies for analytical workloads