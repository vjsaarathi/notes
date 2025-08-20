---
tags:
  - etl-patterns
  - data-ingestion
  - kafka-integration
  - s3-integration
  - postgresql-migration
  - streaming-data
  - batch-processing
---

## ETL Philosophy for ClickHouse

Coming from traditional OLTP systems, I had to rethink ETL for analytical workloads:

**Traditional ETL mindset**: Transform data to fit normalized schema, ensure ACID properties
**ClickHouse ETL mindset**: Transform data for analytical access patterns, optimize for bulk loading, embrace eventual consistency

**Key insight**: ClickHouse ETL is about **feeding the analytical engine efficiently**, not maintaining transactional consistency.

## Data Ingestion Patterns Overview

```mermaid
graph TD
    subgraph "Data Sources"
        A[OLTP Databases<br/>PostgreSQL, MySQL]
        B[Streaming Events<br/>Kafka, Kinesis]  
        C[File Systems<br/>S3, HDFS, Local]
        D[APIs & Services<br/>REST, GraphQL]
    end
    
    subgraph "Ingestion Methods"
        E[Batch ETL<br/>Scheduled extracts]
        F[Real-time Streaming<br/>Event-driven]
        G[Micro-batch<br/>Small frequent loads]
        H[Bulk Import<br/>One-time migrations]
    end
    
    subgraph "ClickHouse Landing"
        I[Raw Tables<br/>Exact source replica]
        J[Staging Tables<br/>Intermediate processing]
        K[Analytical Tables<br/>Optimized for queries]
        L[Materialized Views<br/>Real-time aggregation]
    end
    
    A --> E --> I
    B --> F --> I
    C --> G --> J
    D --> H --> K
    I --> J --> K --> L
```

## PostgreSQL to ClickHouse Migration

This was my most common integration pattern - moving analytical workloads from PostgreSQL to ClickHouse.

### Assessment and Planning Phase

**Step 1: Workload Analysis**
```sql
-- Analyze PostgreSQL query patterns to identify analytical workloads
-- Run this on your PostgreSQL instance:

-- Find analytical queries (large table scans, aggregations)
SELECT 
    query,
    calls,
    total_time,
    mean_time,
    rows / calls as avg_rows_per_call,
    100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0) AS hit_percent
FROM pg_stat_statements 
WHERE rows / calls > 10000  -- Queries processing >10K rows on average
   OR calls < 100           -- Infrequent but complex queries
ORDER BY total_time DESC
LIMIT 20;

-- Identify heavy tables (candidates for ClickHouse)
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size,
    n_tup_ins + n_tup_upd + n_tup_del as total_writes,
    n_tup_upd + n_tup_del as modifications,
    round(100.0 * (n_tup_upd + n_tup_del) / nullif(n_tup_ins + n_tup_upd + n_tup_del, 0), 2) as modification_ratio
FROM pg_stat_user_tables
WHERE n_tup_ins + n_tup_upd + n_tup_del > 1000
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

**Step 2: Data Volume and Growth Analysis**
```sql
-- Historical growth analysis (if you have table stats history)
SELECT 
    table_name,
    current_size_gb,
    daily_growth_gb,
    projected_yearly_size_gb
FROM table_growth_analysis;

-- Query frequency and performance analysis
SELECT 
    extract(hour from query_start) as hour_of_day,
    count(*) as query_count,
    avg(duration) as avg_duration_ms
FROM query_history 
WHERE table_name IN ('orders', 'events', 'transactions')  -- Your analytical tables
GROUP BY hour_of_day
ORDER BY hour_of_day;
```

### Schema Translation Strategy

**PostgreSQL → ClickHouse Type Mapping**:
```sql
-- PostgreSQL schema
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,           -- Auto-incrementing
    customer_id INTEGER NOT NULL,
    order_date DATE NOT NULL,
    order_timestamp TIMESTAMP DEFAULT NOW(),
    total_amount DECIMAL(10,2) NOT NULL,
    order_status VARCHAR(20) DEFAULT 'pending',
    customer_notes TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- ClickHouse equivalent (optimized for analytics)
CREATE TABLE orders (
    order_id UInt32,                       -- No auto-increment needed
    customer_id UInt32,                    -- Unsigned for better compression
    order_date Date,                       -- Same
    order_timestamp DateTime DEFAULT now(), -- ClickHouse function
    total_amount Decimal(10,2),            -- Same precision
    order_status LowCardinality(String) DEFAULT 'pending',  -- Categorical optimization
    customer_notes String,                 -- TEXT → String
    created_at DateTime DEFAULT now(),
    updated_at DateTime DEFAULT now()
) ENGINE = MergeTree()
ORDER BY (order_date, customer_id, order_id)  -- Analytical primary key
PARTITION BY toYYYYMM(order_date);             -- Time-based partitioning
```

### Initial Data Migration

**Approach 1: Direct Migration (Small-Medium Datasets)**
```python
# Python script for direct PostgreSQL → ClickHouse migration
import psycopg2
import clickhouse_connect
from datetime import datetime, timedelta

def migrate_table_batch(table_name, batch_size=100000):
    """Migrate PostgreSQL table to ClickHouse in batches"""
    
    # PostgreSQL connection
    pg_conn = psycopg2.connect("postgresql://user:pass@localhost/source_db")
    pg_cursor = pg_conn.cursor()
    
    # ClickHouse connection
    ch_client = clickhouse_connect.get_client(
        host='clickhouse-server', 
        port=8123,
        database='analytics'
    )
    
    # Get total row count for progress tracking
    pg_cursor.execute(f"SELECT COUNT(*) FROM {table_name}")
    total_rows = pg_cursor.fetchone()[0]
    print(f"Migrating {total_rows:,} rows from {table_name}")
    
    offset = 0
    batch_num = 0
    
    while offset < total_rows:
        batch_num += 1
        print(f"Processing batch {batch_num}, rows {offset:,} to {offset + batch_size:,}")
        
        # Extract batch from PostgreSQL
        pg_cursor.execute(f"""
            SELECT order_id, customer_id, order_date, order_timestamp, 
                   total_amount, order_status, customer_notes, created_at, updated_at
            FROM {table_name}
            ORDER BY order_id
            LIMIT {batch_size} OFFSET {offset}
        """)
        
        rows = pg_cursor.fetchall()
        if not rows:
            break
            
        # Transform data for ClickHouse  
        transformed_rows = []
        for row in rows:
            transformed_row = (
                row[0],  # order_id
                row[1],  # customer_id  
                row[2],  # order_date
                row[3],  # order_timestamp
                float(row[4]),  # total_amount (Decimal → float for ClickHouse)
                row[5] if row[5] else 'pending',  # order_status with default
                row[6] if row[6] else '',  # customer_notes
                row[7],  # created_at
                row[8]   # updated_at
            )
            transformed_rows.append(transformed_row)
        
        # Load into ClickHouse
        ch_client.insert(
            table=table_name,
            data=transformed_rows,
            column_names=['order_id', 'customer_id', 'order_date', 'order_timestamp', 
                         'total_amount', 'order_status', 'customer_notes', 'created_at', 'updated_at']
        )
        
        offset += batch_size
        
    print(f"Migration complete: {total_rows:,} rows migrated")

# Execute migration
migrate_table_batch('orders', batch_size=50000)
```

**Approach 2: CSV Export/Import (Large Datasets)**
```bash
#!/bin/bash
# Export from PostgreSQL to CSV
psql -h source-pg-server -d source_db -c "
COPY (
    SELECT order_id, customer_id, order_date, order_timestamp, 
           total_amount, order_status, customer_notes, created_at, updated_at
    FROM orders 
    ORDER BY order_date
) TO STDOUT WITH CSV HEADER
" > orders_export.csv

# Import to ClickHouse
clickhouse-client --host clickhouse-server --query="
INSERT INTO orders FORMAT CSV
" < orders_export.csv

echo "Migration completed: $(wc -l < orders_export.csv) rows"
```

### Ongoing Synchronization Strategies

**Strategy 1: Timestamp-Based Incremental Sync**
```python
def incremental_sync(table_name, timestamp_column='updated_at'):
    """Sync changes since last sync based on timestamp"""
    
    # Get last sync timestamp from ClickHouse
    last_sync = ch_client.query(f"""
        SELECT max({timestamp_column}) as last_sync 
        FROM {table_name}
    """).first_row[0]
    
    if not last_sync:
        last_sync = datetime(2020, 1, 1)  # Initial sync date
    
    print(f"Syncing changes since: {last_sync}")
    
    # Extract new/updated records from PostgreSQL  
    pg_cursor.execute(f"""
        SELECT * FROM {table_name}
        WHERE {timestamp_column} > %s
        ORDER BY {timestamp_column}
    """, (last_sync,))
    
    new_records = pg_cursor.fetchall()
    print(f"Found {len(new_records)} new/updated records")
    
    if new_records:
        # Insert/replace in ClickHouse (using ReplacingMergeTree)
        ch_client.insert(table_name, new_records)
        print(f"Synced {len(new_records)} records")

# Run incremental sync every 15 minutes
import schedule
schedule.every(15).minutes.do(incremental_sync, 'orders')
```

**Strategy 2: Change Data Capture (CDC)**
```python
# Using Debezium/Kafka for real-time CDC
# This captures all changes from PostgreSQL transaction log

def setup_cdc_pipeline():
    """Setup CDC from PostgreSQL to ClickHouse via Kafka"""
    
    # Debezium connector configuration
    debezium_config = {
        "name": "postgres-orders-connector",
        "config": {
            "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
            "database.hostname": "postgres-server",
            "database.port": "5432", 
            "database.user": "debezium_user",
            "database.password": "password",
            "database.dbname": "source_db",
            "database.server.name": "orders_cdc",
            "table.whitelist": "public.orders,public.customers",
            "slot.name": "clickhouse_replication",
            "plugin.name": "pgoutput"
        }
    }
    
    # ClickHouse Kafka consumer
    kafka_consumer_sql = """
    CREATE TABLE orders_kafka_queue (
        order_id UInt32,
        customer_id UInt32,
        order_date Date,
        total_amount Decimal(10,2),
        operation String  -- 'INSERT', 'UPDATE', 'DELETE'
    ) ENGINE = Kafka()
    SETTINGS 
        kafka_broker_list = 'kafka-broker:9092',
        kafka_topic_list = 'orders_cdc.public.orders',
        kafka_group_name = 'clickhouse_orders_consumer',
        kafka_format = 'JSONEachRow';
    
    -- Materialized view to process CDC events
    CREATE MATERIALIZED VIEW orders_cdc_processor TO orders AS
    SELECT 
        order_id, customer_id, order_date, total_amount
    FROM orders_kafka_queue
    WHERE operation IN ('INSERT', 'UPDATE');
    """
    
    return debezium_config, kafka_consumer_sql
```

## Kafka Streaming Integration

Real-time event streaming was critical for my user analytics and monitoring use cases.

### Kafka Table Engine Setup

```sql
-- Create Kafka source table (acts as queue consumer)
CREATE TABLE user_events_kafka (
    user_id UInt32,
    event_type LowCardinality(String),
    event_timestamp DateTime,
    session_id String,
    page_url String,
    properties String  -- JSON as string
) ENGINE = Kafka()
SETTINGS
    kafka_broker_list = 'kafka-broker-1:9092,kafka-broker-2:9092,kafka-broker-3:9092',
    kafka_topic_list = 'user_events',
    kafka_group_name = 'clickhouse_analytics_consumer',
    kafka_format = 'JSONEachRow',
    kafka_num_consumers = 3,                    -- Parallel consumers
    kafka_max_block_size = 65536,              -- Batch size
    kafka_skip_broken_messages = 1000;         -- Skip malformed messages

-- Create target analytical table
CREATE TABLE user_events (
    user_id UInt32,
    event_type LowCardinality(String),
    event_timestamp DateTime,
    session_id String,
    page_url String,
    properties String,
    -- Derived columns for analysis
    event_date Date MATERIALIZED toDate(event_timestamp),
    event_hour UInt8 MATERIALIZED toHour(event_timestamp)
) ENGINE = MergeTree()
ORDER BY (event_date, user_id, event_timestamp)
PARTITION BY toYYYYMM(event_date);

-- Materialized view to process streaming data
CREATE MATERIALIZED VIEW user_events_consumer TO user_events AS
SELECT 
    user_id,
    event_type,
    event_timestamp,
    session_id,
    page_url,
    properties
FROM user_events_kafka;
```

### Advanced Kafka Configuration

```sql
-- High-throughput configuration
CREATE TABLE high_volume_events_kafka (
    -- Schema definition
    event_id String,
    timestamp DateTime64(3),
    payload String
) ENGINE = Kafka()
SETTINGS
    kafka_broker_list = 'kafka:9092',
    kafka_topic_list = 'high_volume_topic',
    kafka_group_name = 'clickhouse_high_volume',
    kafka_format = 'JSONEachRow',
    
    -- Performance tuning
    kafka_num_consumers = 8,                    -- More parallel consumers
    kafka_max_block_size = 1048576,            -- 1M rows per batch  
    kafka_flush_interval_ms = 5000,            -- 5 second batches
    kafka_poll_timeout_ms = 5000,
    
    -- Error handling
    kafka_skip_broken_messages = 10000,        -- Skip malformed messages
    input_format_allow_errors_num = 1000,      -- Allow parsing errors
    input_format_allow_errors_ratio = 0.1;     -- 10% error tolerance

-- Monitor Kafka consumer performance
SELECT 
    database,
    table,
    name,
    value,
    description
FROM system.metrics
WHERE name LIKE '%Kafka%';
```

### Real-time Analytics with Kafka

```sql
-- Real-time aggregations from streaming data
CREATE MATERIALIZED VIEW real_time_metrics_mv
ENGINE = SummingMergeTree()
ORDER BY (minute, event_type)
AS SELECT
    toStartOfMinute(event_timestamp) as minute,
    event_type,
    count() as event_count,
    uniq(user_id) as unique_users,
    uniq(session_id) as unique_sessions
FROM user_events_kafka
GROUP BY minute, event_type;

-- Query real-time metrics
SELECT 
    minute,
    event_type,
    sum(event_count) as total_events,
    sum(unique_users) as total_users
FROM real_time_metrics_mv
WHERE minute >= now() - INTERVAL 1 HOUR
GROUP BY minute, event_type
ORDER BY minute DESC, total_events DESC;

-- Real-time alerting
SELECT 
    event_type,
    sum(event_count) as events_last_5min
FROM real_time_metrics_mv
WHERE minute >= now() - INTERVAL 5 MINUTE
GROUP BY event_type
HAVING events_last_5min < 100;  -- Alert if events drop below threshold
```

## S3 Integration Patterns

S3 integration became essential for my data lake architectures and large-scale batch processing.

### Direct S3 Querying

```sql
-- Query Parquet files directly from S3
SELECT 
    order_date,
    product_category, 
    sum(total_amount) as revenue
FROM s3(
    'https://my-data-lake.s3.amazonaws.com/orders/year=2024/month=01/*.parquet',
    'AWS_ACCESS_KEY_ID',
    'AWS_SECRET_ACCESS_KEY', 
    'Parquet'
)
WHERE order_date >= '2024-01-01'
GROUP BY order_date, product_category
ORDER BY order_date, revenue DESC;

-- Query CSV files with schema inference
SELECT *
FROM s3(
    'https://my-bucket.s3.amazonaws.com/exports/daily_sales_*.csv',
    'AWS_ACCESS_KEY_ID',
    'AWS_SECRET_ACCESS_KEY',
    'CSVWithNames'  -- First row contains headers
)
LIMIT 10;

-- Query JSON files  
SELECT 
    event_type,
    count() as event_count
FROM s3(
    'https://logs-bucket.s3.amazonaws.com/events/2024/01/15/*.json',
    'AWS_ACCESS_KEY_ID', 
    'AWS_SECRET_ACCESS_KEY',
    'JSONEachRow'
)
GROUP BY event_type;
```

### S3 Table Engine for External Data

```sql
-- Create table backed by S3 data
CREATE TABLE s3_order_history (
    order_id UInt32,
    customer_id UInt32,
    order_date Date,
    total_amount Decimal(10,2)
) ENGINE = S3(
    'https://historical-data.s3.amazonaws.com/orders/{_partition_id}.parquet',
    'AWS_ACCESS_KEY_ID',
    'AWS_SECRET_ACCESS_KEY',
    'Parquet'
)
PARTITION BY toYYYYMM(order_date);

-- Query S3 table like any ClickHouse table
SELECT 
    toStartOfMonth(order_date) as month,
    count() as orders,
    sum(total_amount) as revenue
FROM s3_order_history
WHERE order_date >= '2023-01-01'
GROUP BY month
ORDER BY month;
```

### Data Export to S3

```sql
-- Export analytical results to S3
INSERT INTO FUNCTION s3(
    'https://exports-bucket.s3.amazonaws.com/monthly_reports/revenue_2024_01.parquet',
    'AWS_ACCESS_KEY_ID',
    'AWS_SECRET_ACCESS_KEY', 
    'Parquet'
)
SELECT 
    product_category,
    sum(total_amount) as monthly_revenue,
    count() as order_count,
    avg(total_amount) as avg_order_value
FROM orders
WHERE toYYYYMM(order_date) = 202401
GROUP BY product_category;

-- Scheduled export job (using cron + clickhouse-client)
-- Export daily aggregations every morning at 1 AM
# 0 1 * * * /usr/bin/clickhouse-client --query="INSERT INTO FUNCTION s3('https://exports.s3.amazonaws.com/daily/$(date +%Y-%m-%d).csv', 'key', 'secret', 'CSV') SELECT * FROM daily_summary WHERE date = yesterday()"
```

### S3 Data Lake Integration

```python
# Python script for S3 data lake ETL
import boto3
import pandas as pd
import clickhouse_connect
from datetime import datetime, timedelta

class S3DataLakeETL:
    def __init__(self):
        self.s3_client = boto3.client('s3')
        self.ch_client = clickhouse_connect.get_client(host='clickhouse-server')
        
    def process_daily_raw_data(self, date):
        """Process raw JSON logs from S3 into structured ClickHouse tables"""
        
        # List S3 objects for the date
        bucket = 'raw-logs-bucket'
        prefix = f'events/{date.strftime("%Y/%m/%d")}/'
        
        objects = self.s3_client.list_objects_v2(
            Bucket=bucket,
            Prefix=prefix
        )
        
        all_events = []
        
        for obj in objects.get('Contents', []):
            key = obj['Key']
            print(f"Processing {key}")
            
            # Read JSON file from S3
            response = self.s3_client.get_object(Bucket=bucket, Key=key)
            content = response['Body'].read().decode('utf-8')
            
            # Parse JSON lines
            for line in content.strip().split('\n'):
                try:
                    event = json.loads(line)
                    processed_event = self.transform_event(event)
                    all_events.append(processed_event)
                except Exception as e:
                    print(f"Error processing line: {e}")
                    continue
        
        # Bulk insert to ClickHouse
        if all_events:
            self.ch_client.insert('processed_events', all_events)
            print(f"Inserted {len(all_events)} events for {date}")
            
    def transform_event(self, raw_event):
        """Transform raw event JSON to ClickHouse schema"""
        return {
            'event_id': raw_event.get('id', ''),
            'user_id': int(raw_event.get('user_id', 0)),
            'event_type': raw_event.get('type', 'unknown'),
            'timestamp': datetime.fromisoformat(raw_event.get('timestamp', '')),
            'properties': json.dumps(raw_event.get('properties', {}))
        }
    
    def run_daily_etl(self):
        """Run ETL for yesterday's data"""
        yesterday = datetime.now() - timedelta(days=1)
        self.process_daily_raw_data(yesterday.date())

# Schedule daily ETL
etl = S3DataLakeETL()
etl.run_daily_etl()
```

## File-Based Integration

### CSV/TSV File Processing

```sql
-- Import CSV files with flexible schema
INSERT INTO orders SELECT * FROM file('orders_2024_01.csv', 'CSV') 
SETTINGS 
    input_format_allow_errors_num = 1000,
    input_format_allow_errors_ratio = 0.1;

-- Process multiple files with patterns
INSERT INTO events SELECT * FROM file('events_*.json', 'JSONEachRow');

-- Import with data transformation
INSERT INTO clean_orders
SELECT 
    order_id,
    customer_id,
    parseDateTime64BestEffort(order_date_string) as order_date,
    toDecimal64(amount_string, 2) as total_amount
FROM file('messy_orders.csv', 'CSVWithNames');
```

### Excel File Integration

```python
# Python script for Excel to ClickHouse ETL
import pandas as pd
import clickhouse_connect

def excel_to_clickhouse(excel_file, sheet_name, table_name):
    """Convert Excel sheet to ClickHouse table"""
    
    # Read Excel file
    df = pd.read_excel(excel_file, sheet_name=sheet_name)
    
    # Clean data
    df = df.dropna()  # Remove empty rows
    df.columns = [col.lower().replace(' ', '_') for col in df.columns]  # Clean column names
    
    # Data type optimization for ClickHouse
    for col in df.columns:
        if df[col].dtype == 'object':
            # Check if it's a categorical column (few unique values)
            if df[col].nunique() / len(df) < 0.1:  # Less than 10% unique values
                df[col] = df[col].astype('category')
    
    # Connect to ClickHouse
    client = clickhouse_connect.get_client(host='clickhouse-server')
    
    # Insert data
    client.insert_df(table_name, df)
    print(f"Inserted {len(df)} rows into {table_name}")

# Usage
excel_to_clickhouse('sales_report.xlsx', 'Monthly Sales', 'monthly_sales')
```

## API Integration Patterns

### REST API Data Ingestion

```python
import requests
import clickhouse_connect
from datetime import datetime, timedelta
import time

class APIDataIngestion:
    def __init__(self, api_base_url, ch_host):
        self.api_base_url = api_base_url
        self.ch_client = clickhouse_connect.get_client(host=ch_host)
        
    def fetch_api_data(self, endpoint, params=None):
        """Fetch data from REST API with pagination"""
        all_data = []
        page = 1
        
        while True:
            params_with_page = (params or {}).copy()
            params_with_page.update({'page': page, 'limit': 1000})
            
            response = requests.get(f"{self.api_base_url}/{endpoint}", params=params_with_page)
            response.raise_for_status()
            
            data = response.json()
            
            if not data.get('results'):
                break
                
            all_data.extend(data['results'])
            
            if len(data['results']) < 1000:  # Last page
                break
                
            page += 1
            time.sleep(0.1)  # Rate limiting
            
        return all_data
    
    def sync_customer_data(self):
        """Sync customer data from CRM API"""
        
        # Get last sync timestamp
        last_sync_result = self.ch_client.query(
            "SELECT max(updated_at) as last_sync FROM customers"
        )
        last_sync = last_sync_result.first_row[0] if last_sync_result.row_count > 0 else None
        
        # Fetch updated customers
        params = {}
        if last_sync:
            params['updated_since'] = last_sync.isoformat()
            
        customers = self.fetch_api_data('customers', params)
        
        if customers:
            # Transform API data to ClickHouse schema
            transformed = []
            for customer in customers:
                transformed.append({
                    'customer_id': customer['id'],
                    'customer_name': customer['name'],
                    'email': customer['email'],
                    'customer_segment': customer.get('segment', 'standard'),
                    'created_at': datetime.fromisoformat(customer['created_at']),
                    'updated_at': datetime.fromisoformat(customer['updated_at'])
                })
            
            # Use ReplacingMergeTree for upsert behavior
            self.ch_client.insert('customers', transformed)
            print(f"Synced {len(customers)} customers")

# Schedule API sync every hour
api_sync = APIDataIngestion('https://api.mycompany.com/v1', 'clickhouse-server')
api_sync.sync_customer_data()
```

## Data Quality and Validation

### Data Quality Checks

```sql
-- Create data quality monitoring table
CREATE TABLE data_quality_checks (
    check_date Date,
    table_name String,
    check_type String,
    check_description String,
    expected_value String,
    actual_value String,
    status Enum8('PASS'=1, 'FAIL'=2, 'WARN'=3),
    created_at DateTime DEFAULT now()
) ENGINE = MergeTree()
ORDER BY (check_date, table_name, check_type);

-- Row count validation
INSERT INTO data_quality_checks
SELECT 
    today() as check_date,
    'orders' as table_name,
    'row_count' as check_type,
    'Daily order count should be > 1000' as check_description,
    '>1000' as expected_value,
    toString(count()) as actual_value,
    multiIf(count() > 1000, 'PASS', count() > 500, 'WARN', 'FAIL') as status
FROM orders
WHERE order_date = today();

-- Data freshness validation  
INSERT INTO data_quality_checks
SELECT 
    today() as check_date,
    'user_events' as table_name,
    'data_freshness' as check_type,
    'Latest event should be within last hour' as check_description,
    '<1 hour' as expected_value,
    toString(dateDiff('minute', max_timestamp, now())) || ' minutes' as actual_value,
    multiIf(max_timestamp > now() - INTERVAL 1 HOUR, 'PASS', 'FAIL') as status
FROM (SELECT max(event_timestamp) as max_timestamp FROM user_events);

-- Null value validation
INSERT INTO data_quality_checks  
SELECT 
    today() as check_date,
    'customers' as table_name,
    'null_check' as check_type,
    'Customer email should not be null' as check_description,
    '0' as expected_value,
    toString(countIf(email = '')) as actual_value,
    multiIf(countIf(email = '') = 0, 'PASS', 'FAIL') as status
FROM customers;
```

### ETL Monitoring Dashboard

```sql
-- ETL performance monitoring
CREATE MATERIALIZED VIEW etl_performance_mv
ENGINE = SummingMergeTree()
ORDER BY (date, job_name)
AS SELECT
    toDate(start_time) as date,
    job_name,
    count() as job_runs,
    sum(rows_processed) as total_rows,
    sum(duration_seconds) as total_duration,
    avg(duration_seconds) as avg_duration
FROM etl_job_log
GROUP BY date, job_name;

-- Query ETL dashboard data
SELECT 
    job_name,
    sum(job_runs) as runs_last_7_days,
    sum(total_rows) as rows_processed,
    avg(avg_duration) as avg_runtime_seconds,
    max(case when date = today() then total_rows else 0 end) as rows_today
FROM etl_performance_mv
WHERE date >= today() - INTERVAL 7 DAY
GROUP BY job_name
ORDER BY runs_last_7_days DESC;
```

## ETL Best Practices and Patterns

### Idempotent ETL Design

```python
def idempotent_daily_etl(date):
    """ETL that can be safely re-run for same date"""
    
    # Clear existing data for the date (if re-running)
    ch_client.command(f"""
        ALTER TABLE daily_aggregates 
        DELETE WHERE date = '{date}'
    """)
    
def idempotent_daily_etl(date):
    """ETL that can be safely re-run for same date"""
    
    # Clear existing data for the date (if re-running)
    ch_client.command(f"""
        ALTER TABLE daily_aggregates 
        DELETE WHERE date = '{date}'
    """)
    
    # Process data for the specific date
    result = ch_client.query(f"""
        INSERT INTO daily_aggregates
        SELECT 
            '{date}' as date,
            product_category,
            sum(total_amount) as daily_revenue,
            count() as order_count,
            uniq(customer_id) as unique_customers
        FROM orders
        WHERE toDate(order_timestamp) = '{date}'
        GROUP BY product_category
    """)
    
    print(f"Processed {result.summary['written_rows']} aggregates for {date}")

### Error Handling and Recovery

```python
import logging
from contextlib import contextmanager

class ETLJobRunner:
    def __init__(self, job_name):
        self.job_name = job_name
        self.logger = logging.getLogger(job_name)
        
    @contextmanager
    def job_execution_context(self, date):
        """Context manager for ETL job execution with logging and error handling"""
        job_id = f"{self.job_name}_{date}"
        start_time = datetime.now()
        
        # Log job start
        self.ch_client.insert('etl_job_log', [{
            'job_id': job_id,
            'job_name': self.job_name,
            'start_time': start_time,
            'status': 'RUNNING',
            'date_processed': date
        }])
        
        try:
            self.logger.info(f"Starting ETL job {job_id}")
            yield
            
            # Log successful completion
            duration = (datetime.now() - start_time).total_seconds()
            self.ch_client.insert('etl_job_log', [{
                'job_id': job_id,
                'job_name': self.job_name,
                'start_time': start_time,
                'end_time': datetime.now(),
                'duration_seconds': duration,
                'status': 'SUCCESS',
                'date_processed': date
            }])
            
            self.logger.info(f"ETL job {job_id} completed successfully in {duration:.2f}s")
            
        except Exception as e:
            # Log failure
            duration = (datetime.now() - start_time).total_seconds()
            error_message = str(e)
            
            self.ch_client.insert('etl_job_log', [{
                'job_id': job_id,
                'job_name': self.job_name,
                'start_time': start_time,
                'end_time': datetime.now(),
                'duration_seconds': duration,
                'status': 'FAILED',
                'error_message': error_message,
                'date_processed': date
            }])
            
            self.logger.error(f"ETL job {job_id} failed after {duration:.2f}s: {error_message}")
            
            # Send alert
            self.send_failure_alert(job_id, error_message)
            raise
    
    def send_failure_alert(self, job_id, error_message):
        """Send alert for job failure (Slack, email, etc.)"""
        # Slack notification example
        import requests
        slack_webhook = "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"
        
        message = {
            "text": f"🚨 ETL Job Failed: {job_id}",
            "attachments": [{
                "color": "danger",
                "fields": [
                    {"title": "Job", "value": self.job_name, "short": True},
                    {"title": "Error", "value": error_message, "short": False}
                ]
            }]
        }
        
        requests.post(slack_webhook, json=message)

# Usage
etl_runner = ETLJobRunner('daily_revenue_aggregation')
with etl_runner.job_execution_context('2024-01-15'):
    idempotent_daily_etl('2024-01-15')
```

### Batch Size Optimization

```python
def optimal_batch_size_finder(table_name, test_sizes=[1000, 5000, 10000, 50000, 100000]):
    """Find optimal batch size for specific table"""
    
    results = []
    
    for batch_size in test_sizes:
        print(f"Testing batch size: {batch_size}")
        
        # Create test data
        test_data = generate_test_data(batch_size)
        
        # Measure insertion performance
        start_time = time.time()
        ch_client.insert(table_name, test_data)
        duration = time.time() - start_time
        
        rows_per_second = batch_size / duration
        
        results.append({
            'batch_size': batch_size,
            'duration': duration,
            'rows_per_second': rows_per_second
        })
        
        print(f"  Duration: {duration:.2f}s, Rate: {rows_per_second:,.0f} rows/sec")
    
    # Find optimal batch size
    optimal = max(results, key=lambda x: x['rows_per_second'])
    print(f"\nOptimal batch size: {optimal['batch_size']:,} ({optimal['rows_per_second']:,.0f} rows/sec)")
    
    return optimal['batch_size']

# Find and use optimal batch size
optimal_batch = optimal_batch_size_finder('orders')
```

### Memory-Efficient Data Processing

```python
def memory_efficient_etl(source_query, target_table, batch_size=50000):
    """Process large datasets without loading everything into memory"""
    
    offset = 0
    total_processed = 0
    
    while True:
        # Fetch batch with LIMIT/OFFSET
        batch_query = f"""
        {source_query}
        ORDER BY primary_key_column
        LIMIT {batch_size} OFFSET {offset}
        """
        
        batch_data = source_db.execute(batch_query).fetchall()
        
        if not batch_data:
            break
            
        # Transform batch
        transformed_batch = []
        for row in batch_data:
            transformed_row = transform_row(row)  # Your transformation logic
            transformed_batch.append(transformed_row)
        
        # Insert batch to ClickHouse
        ch_client.insert(target_table, transformed_batch)
        
        total_processed += len(batch_data)
        offset += batch_size
        
        print(f"Processed {total_processed:,} rows...")
        
        # Memory cleanup
        del batch_data, transformed_batch
        
    print(f"ETL complete: {total_processed:,} total rows processed")
```

## Performance Optimization Patterns

### Parallel Processing

```python
import concurrent.futures
from multiprocessing import cpu_count

def parallel_etl_by_partition(date_range, max_workers=None):
    """Process multiple date partitions in parallel"""
    
    if max_workers is None:
        max_workers = min(cpu_count(), 8)  # Don't overwhelm the system
    
    def process_single_date(date):
        return idempotent_daily_etl(date)
    
    with concurrent.futures.ThreadPoolExecutor(max_workers=max_workers) as executor:
        # Submit all date processing jobs
        future_to_date = {
            executor.submit(process_single_date, date): date 
            for date in date_range
        }
        
        # Collect results
        for future in concurrent.futures.as_completed(future_to_date):
            date = future_to_date[future]
            try:
                result = future.result()
                print(f"Successfully processed {date}")
            except Exception as e:
                print(f"Error processing {date}: {e}")

# Process last 30 days in parallel
date_range = [datetime.now().date() - timedelta(days=i) for i in range(30)]
parallel_etl_by_partition(date_range)
```

### ClickHouse-Specific Optimizations

```sql
-- Optimize INSERT settings for bulk loading
SET max_insert_block_size = 1048576;        -- 1M rows per block
SET min_insert_block_size_rows = 1000000;   -- Don't create tiny parts
SET max_insert_threads = 16;                -- Parallel insertion
SET insert_quorum = 2;                      -- Wait for replication (if using replicas)

-- Optimize SELECT settings for ETL extracts
SET max_threads = 16;                       -- Use all available cores
SET max_memory_usage = 20000000000;         -- 20GB memory limit for large extracts
SET max_bytes_before_external_sort = 10000000000;  -- 10GB before external sort

-- Batch processing with optimal settings
INSERT INTO target_table
SELECT 
    -- Complex transformations
    CASE WHEN condition THEN value1 ELSE value2 END as transformed_column,
    -- Aggregations within INSERT
    sum(amount) OVER (PARTITION BY customer_id) as customer_total
FROM source_table
WHERE date_column >= '2024-01-01'
SETTINGS 
    optimize_read_in_order = 1,             -- Optimize ORDER BY reads
    optimize_aggregation_in_order = 1,      -- Optimize GROUP BY
    max_threads = 16;
```

### Data Deduplication Strategies

```sql
-- Strategy 1: Use ReplacingMergeTree with version column
CREATE TABLE events_deduplicated (
    event_id String,
    user_id UInt32,
    event_type String,
    timestamp DateTime,
    version UInt64 DEFAULT now64()          -- Automatic versioning
) ENGINE = ReplacingMergeTree(version)
ORDER BY (event_id, user_id);

-- Strategy 2: Manual deduplication during ETL
INSERT INTO clean_events
SELECT 
    event_id,
    user_id,
    event_type,
    timestamp,
    argMax(properties, timestamp) as latest_properties  -- Keep latest version
FROM raw_events
WHERE processing_date = today()
GROUP BY event_id, user_id, event_type, timestamp;

-- Strategy 3: Use DISTINCT for simple deduplication
INSERT INTO unique_users
SELECT DISTINCT 
    user_id,
    first_name,
    last_name,
    email
FROM user_registrations
WHERE registration_date = yesterday();
```

## Integration Monitoring and Alerting

### Comprehensive Monitoring Setup

```sql
-- Create monitoring tables
CREATE TABLE integration_health (
    check_timestamp DateTime DEFAULT now(),
    integration_name String,
    source_system String,
    target_table String,
    
    -- Health metrics
    records_processed UInt64,
    processing_duration_seconds UInt32,
    error_count UInt32,
    data_lag_minutes UInt32,
    
    -- Status
    status Enum8('HEALTHY'=1, 'WARNING'=2, 'ERROR'=3),
    error_message String
) ENGINE = MergeTree()
ORDER BY (check_timestamp, integration_name)
TTL check_timestamp + INTERVAL 30 DAY DELETE;  -- Cleanup old monitoring data

-- Kafka consumer lag monitoring
INSERT INTO integration_health
SELECT 
    now() as check_timestamp,
    'kafka_user_events' as integration_name,
    'kafka' as source_system,
    'user_events' as target_table,
    
    -- Calculate lag from Kafka metadata
    consumer_offset - high_water_mark as records_behind,
    0 as processing_duration_seconds,
    0 as error_count,
    dateDiff('minute', max_timestamp, now()) as data_lag_minutes,
    
    multiIf(
        records_behind > 10000, 'ERROR',
        records_behind > 1000, 'WARNING', 
        'HEALTHY'
    ) as status,
    
    case when records_behind > 10000 
         then concat('High lag: ', toString(records_behind), ' messages behind')
         else '' 
    end as error_message
    
FROM kafka_consumer_metadata;
```

### Real-time Alerting System

```python
class IntegrationMonitor:
    def __init__(self):
        self.ch_client = clickhouse_connect.get_client(host='clickhouse-server')
        
    def check_data_freshness(self, table_name, max_lag_minutes=30):
        """Check if data in table is fresh enough"""
        
        result = self.ch_client.query(f"""
            SELECT 
                max(timestamp_column) as max_timestamp,
                dateDiff('minute', max_timestamp, now()) as lag_minutes
            FROM {table_name}
        """)
        
        if result.row_count == 0:
            return {'status': 'ERROR', 'message': 'No data found'}
            
        lag_minutes = result.first_row[1]
        
        if lag_minutes > max_lag_minutes:
            return {
                'status': 'ERROR', 
                'message': f'Data is {lag_minutes} minutes old (max: {max_lag_minutes})'
            }
        elif lag_minutes > max_lag_minutes * 0.8:
            return {
                'status': 'WARNING',
                'message': f'Data is {lag_minutes} minutes old (approaching limit)'
            }
        else:
            return {'status': 'HEALTHY', 'message': f'Data is {lag_minutes} minutes old'}
    
    def check_row_count_anomaly(self, table_name, date_column, expected_min_rows=1000):
        """Detect anomalies in daily row counts"""
        
        result = self.ch_client.query(f"""
            SELECT 
                toDate({date_column}) as date,
                count() as row_count,
                avg(count()) OVER (ORDER BY toDate({date_column}) 
                                 ROWS BETWEEN 7 PRECEDING AND 1 PRECEDING) as avg_last_7_days
            FROM {table_name}
            WHERE toDate({date_column}) >= today() - INTERVAL 1 DAY
            GROUP BY toDate({date_column})
            ORDER BY date DESC
            LIMIT 1
        """)
        
        if result.row_count == 0:
            return {'status': 'ERROR', 'message': 'No recent data found'}
            
        row = result.first_row
        today_count = row[1]
        avg_count = row[2] if row[2] else expected_min_rows
        
        if today_count < expected_min_rows:
            return {
                'status': 'ERROR',
                'message': f'Row count too low: {today_count} (min: {expected_min_rows})'
            }
        elif today_count < avg_count * 0.5:  # 50% below average
            return {
                'status': 'WARNING', 
                'message': f'Row count below average: {today_count} vs {avg_count:.0f} avg'
            }
        else:
            return {'status': 'HEALTHY', 'message': f'Row count normal: {today_count}'}
    
    def run_all_checks(self):
        """Run all monitoring checks and send alerts"""
        
        checks = [
            ('user_events', 'event_timestamp', 30, 5000),
            ('orders', 'order_timestamp', 60, 1000), 
            ('page_views', 'timestamp', 15, 10000)
        ]
        
        alerts = []
        
        for table_name, timestamp_col, max_lag, min_rows in checks:
            # Data freshness check
            freshness_result = self.check_data_freshness(table_name, max_lag)
            if freshness_result['status'] != 'HEALTHY':
                alerts.append(f"🚨 {table_name} freshness: {freshness_result['message']}")
            
            # Row count anomaly check  
            count_result = self.check_row_count_anomaly(table_name, timestamp_col, min_rows)
            if count_result['status'] != 'HEALTHY':
                alerts.append(f"📊 {table_name} count: {count_result['message']}")
        
        # Send alerts if any issues found
        if alerts:
            self.send_slack_alert('\n'.join(alerts))
        else:
            print("✅ All integration checks passed")
    
    def send_slack_alert(self, message):
        """Send alert to Slack"""
        webhook_url = "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"
        
        payload = {
            "text": "ClickHouse Integration Alert",
            "attachments": [{
                "color": "danger",
                "text": message,
                "ts": int(time.time())
            }]
        }
        
        requests.post(webhook_url, json=payload)

# Run monitoring checks every 5 minutes
monitor = IntegrationMonitor()
monitor.run_all_checks()
```

## ETL Best Practices Summary

### Design Principles I Follow

1. **Idempotency**: Design ETL jobs to be safely re-runnable
2. **Incremental processing**: Process only new/changed data when possible  
3. **Error isolation**: One bad record shouldn't fail entire batch
4. **Monitoring first**: Instrument before deploying to production
5. **Schema evolution**: Plan for source schema changes

### Performance Optimization Rules

1. **Batch size optimization**: Test to find optimal batch sizes (usually 10K-100K rows)
2. **Parallel processing**: Process multiple partitions/dates simultaneously
3. **Memory management**: Use streaming/batch processing for large datasets
4. **ClickHouse settings**: Tune INSERT and SELECT settings for bulk operations
5. **Compression awareness**: Choose appropriate data types for better compression

### Operational Excellence

1. **Comprehensive logging**: Log start/end times, row counts, errors
2. **Data quality checks**: Validate completeness, freshness, accuracy
3. **Alert fatigue prevention**: Set meaningful thresholds, escalation policies
4. **Recovery procedures**: Document and test failure recovery processes
5. **Capacity planning**: Monitor resource usage and plan for growth

### Integration Pattern Selection

| Pattern | Use Case | Pros | Cons |
|---------|----------|------|------|
| **Batch ETL** | Historical migration, nightly processing | Simple, reliable | Higher latency |
| **Kafka Streaming** | Real-time events, user analytics | Low latency, scalable | Complex setup |
| **CDC** | Database synchronization | Real-time, automatic | Requires source changes |
| **API Polling** | Third-party data, CRM sync | Flexible, external control | API rate limits |
| **File-based** | Data exports, partner feeds | Simple, widely supported | Manual coordination |

**Key insight**: ClickHouse ETL success depends on understanding your data patterns, choosing the right integration approach, and optimizing for bulk analytical processing rather than transactional consistency.
