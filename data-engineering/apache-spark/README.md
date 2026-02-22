
# Apache Spark & Databricks - Complete Reference Guide

> **Author’s Note:** This documentation covers everything discussed about Apache Spark and Databricks for AI/Data Engineering interview preparation and practical implementation.

-----

## Table of Contents

1. [Spark Fundamentals](#spark-fundamentals)
1. [Architecture & Components](#architecture--components)
1. [Delta Lake](#delta-lake)
1. [Performance Optimization](#performance-optimization)
1. [Databricks Platform](#databricks-platform)
1. [Data Quality](#data-quality)
1. [Interview Preparation](#interview-preparation)
1. [Best Practices](#best-practices)

-----

## Spark Fundamentals

### What is Apache Spark?

- Distributed data processing engine
- In-memory computation (100× faster than MapReduce)
- Supports batch and streaming
- APIs: Python (PySpark), Scala, Java, R, SQL

### Installation Options

1. **PySpark via pip** (Easiest for learning)
   
   ```bash
   pip install pyspark findspark
   ```
1. **Docker** (Reproducible)
   
   ```bash
   docker run -p 8888:8888 jupyter/pyspark-notebook
   ```
1. **Manual** (Full control)
- Download from Apache Spark website
- Requires Java 11+
- Set SPARK_HOME environment variable
1. **Cloud Platforms** (Production)
- AWS EMR
- Google Cloud Dataproc
- Azure HDInsight
- Databricks

-----

## Architecture & Components

### Core Components

```
┌─────────────────────────────────┐
│        DRIVER PROGRAM           │
│     SparkContext/Session        │
│  • Creates DAG                  │
│  • Schedules tasks              │
└───────────┬─────────────────────┘
            │
            ▼
    ┌───────────────┐
    │ CLUSTER       │
    │ MANAGER       │
    └───────┬───────┘
            │
    ┌───────┴───────┬───────┐
    ▼               ▼       ▼
┌─────────┐   ┌─────────┐ ┌─────────┐
│EXECUTOR │   │EXECUTOR │ │EXECUTOR │
│  Cache  │   │  Cache  │ │  Cache  │
│ Task 1  │   │ Task 3  │ │ Task 5  │
│ Task 2  │   │ Task 4  │ │ Task 6  │
└─────────┘   └─────────┘ └─────────┘
```

### Key Concepts

**Partition**

- Chunk of data processed independently
- Rule of thumb: 2-4 partitions per CPU core
- Ideal size: 128MB - 1GB per partition

**Shuffle**

- Data movement across executors over network
- Expensive operation (disk I/O, network, serialization)
- Triggered by: `groupBy()`, `join()`, `repartition()`, `orderBy()`

**Executor Core**

- CPU thread within executor that runs one task
- Recommended: 4-5 cores per executor
- More cores = more parallel tasks

**SparkSession vs SparkContext**

- **SparkSession** (Spark 2.0+): Unified entry point for DataFrame/SQL/RDD
- **SparkContext**: Legacy, low-level RDD-only API
- SparkSession contains SparkContext internally

### Repartition vs Coalesce

|Aspect   |`repartition(n)`                       |`coalesce(n)`            |
|---------|---------------------------------------|-------------------------|
|Shuffle  |Yes (expensive)                        |No (cheap)               |
|Direction|Increase or decrease                   |Decrease only            |
|Balance  |Evenly distributed                     |May be uneven            |
|Use Case |Need more partitions or perfect balance|Reduce files before write|

-----

## Delta Lake

### What is Delta Lake?

- Open-source storage layer on top of Parquet
- Provides ACID transactions
- Created by Databricks, now Linux Foundation project

### Key Features

**1. ACID Transactions**

```python
df.write.format("delta").mode("overwrite").save("/data/table")
```

**2. Time Travel**

```python
# Version-based
spark.read.format("delta").option("versionAsOf", 10).load("/data/table")

# Timestamp-based
spark.read.format("delta").option("timestampAsOf", "2024-01-01").load("/data/table")
```

**3. MERGE (Upsert)**

```python
from delta.tables import DeltaTable

deltaTable = DeltaTable.forPath(spark, "/data/users")
deltaTable.alias("target").merge(
    updates.alias("source"),
    "target.id = source.id"
).whenMatchedUpdateAll().whenNotMatchedInsertAll().execute()
```

**4. OPTIMIZE**

- Compacts small files into larger ones
- Improves read performance

**5. Z-ORDER**

- Co-locates related data in same files
- Enables data skipping (faster queries)
- Order columns by cardinality (high to low)

```python
spark.sql("OPTIMIZE delta.`/data/table` ZORDER BY (user_id, date)")
```

**6. VACUUM**

```python
spark.sql("VACUUM delta.`/data/table` RETAIN 168 HOURS")
```

### Delta Lake vs Parquet

|Feature         |Parquet           |Delta Lake|
|----------------|------------------|----------|
|ACID            |✗                 |✓         |
|Time Travel     |✗                 |✓         |
|MERGE/Upsert    |Manual            |Built-in  |
|Schema Evolution|✗                 |✓         |
|OPTIMIZE        |Manual repartition|Built-in  |

### Managed vs External Tables

**`saveAsTable()` - Managed**

- Spark controls data location
- DROP TABLE deletes data
- Use for: Internal analytics

**`save()` - External**

- You control data location
- DROP TABLE keeps data
- Use for: Data lakes, shared storage

-----

## Performance Optimization

### 1. Data Skew

**Problem:** Some partitions have much more data than others

**Detection:**

```python
# Check partition sizes
df.groupBy(spark_partition_id()).count().show()

# Check key distribution
df.groupBy("user_id").count().orderBy(desc("count")).show()
```

**Solution: Salting**

- Add random suffix to hot keys
- Distributes load across executors

```python
# Salt hot keys
df_salted = df.withColumn(
    "salted_key",
    concat(col("user_id"), lit("_"), floor(rand() * 10).cast("int"))
)
```

### 2. Broadcast Join

**Use when:** One table is small (< 100MB)

```python
from pyspark.sql.functions import broadcast

large_df.join(broadcast(small_df), "key")
```

**Benefits:**

- Avoids shuffle of large table
- 10-100× faster for appropriate data sizes

### 3. Executor Sizing

**Formula:**

```
Cores per executor: 4-5 (sweet spot)
Memory per executor: 2-4GB per core (adjust for workload)
Num executors: (total_cores / cores_per_executor) - 1
Partitions: 2-4× total cores
```

**Example Configuration:**

```python
spark.conf.set("spark.executor.instances", "29")
spark.conf.set("spark.executor.cores", "5")
spark.conf.set("spark.executor.memory", "19g")
spark.conf.set("spark.executor.memoryOverhead", "2g")
```

### 4. Troubleshooting Slow Queries

**Check in order:**

1. **Spark UI** - Find bottleneck stage
1. **Query Plan** - Look for shuffles
1. **Data Skew** - Check key distribution
1. **Partitions** - Count and sizes
1. **Memory** - GC time > 10%?
1. **Delta** - Too many small files?

### 5. Common Optimizations

```python
# Enable adaptive query execution
spark.conf.set("spark.sql.adaptive.enabled", "true")

# Cache frequently accessed data
df.cache()

# Filter early (predicate pushdown)
df.filter(col("date") == "2024-01-01").groupBy("user_id").count()

# Avoid collect() on large data
# BAD: df.collect()
# GOOD: df.write.parquet("output")
```

-----

## Databricks Platform

### What is Databricks?

- Unified analytics platform built on Spark
- Founded by Spark creators (2013)
- Cloud-based (AWS, Azure, GCP)
- Managed Spark + Delta Lake + ML + Collaboration

### Key Components

**1. Workspaces**

- Collaborative notebooks
- Jobs & workflows
- Dashboards
- Access control

**2. Cluster Types**

|Type         |Use Case       |Lifecycle     |Cost         |
|-------------|---------------|--------------|-------------|
|All-Purpose  |Interactive dev|Manual        |Higher       |
|Job Cluster  |Automated jobs |Auto-terminate|Lower        |
|SQL Warehouse|BI queries     |Serverless    |Pay-per-query|

**3. Unity Catalog**

- Centralized data governance
- 3-level namespace: `catalog.schema.table`
- Fine-grained access control
- Data lineage tracking

**4. Delta Live Tables (DLT)**

- Declarative ETL framework
- Built-in data quality
- Auto dependency resolution
- Production monitoring

**When to use DLT:**

- ✅ Medallion architecture (Bronze/Silver/Gold)
- ✅ Data quality critical
- ✅ Streaming pipelines
- ✅ Complex dependencies
- ❌ Simple one-off jobs
- ❌ Ad-hoc analysis
- ❌ Need portability (Databricks-only)

### Medallion Architecture

```
BRONZE (Raw)         SILVER (Cleaned)      GOLD (Business)
┌──────────┐        ┌──────────┐          ┌──────────┐
│ JSON     │        │ Validated│          │ Metrics  │
│ CSV      │ ─────> │ Deduped  │ ───────> │ Reports  │
│ Raw logs │        │ Joined   │          │ Features │
└──────────┘        └──────────┘          └──────────┘
Ingest all          Clean & conform       Business ready
No transforms       Apply schema          Aggregated
Append-only         Type casting          Optimized for BI
```

-----

## Data Quality

### Quality Dimensions

1. **Completeness** - No nulls/missing
1. **Accuracy** - Valid ranges
1. **Consistency** - Matches business rules
1. **Uniqueness** - No duplicates
1. **Timeliness** - Fresh data
1. **Validity** - Correct types

### Implementation Approaches

**1. Custom Validation Functions**

```python
def check_completeness(df, columns):
    for col in columns:
        null_count = df.filter(col(col).isNull()).count()
        print(f"{col}: {null_count} nulls")
```

**2. Expectations Framework**

```python
class DataQualityChecker:
    def expect_column_values_not_null(self, column, threshold=1.0)
    def expect_column_values_in_set(self, column, valid_values)
    def expect_column_values_between(self, column, min_val, max_val)
    def expect_column_unique(self, column)
```

**3. Great Expectations** (Industry Standard)

```python
import great_expectations as gx
ge_df = SparkDFDataset(df)
ge_df.expect_column_values_to_not_be_null("user_id")
```

**4. Delta Live Tables**

```python
@dlt.table
@dlt.expect("valid_user", "user_id IS NOT NULL")
@dlt.expect_or_drop("valid_age", "age BETWEEN 0 AND 120")
@dlt.expect_or_fail("critical", "timestamp IS NOT NULL")
def silver_table():
    return dlt.read("bronze_table")
```

### Quarantine Pattern (Recommended)

```
Raw Data
    │
    ├──> VALID DATA ──> Silver/Gold Tables
    │
    └──> INVALID DATA ──> Quarantine Table (for review)
```

-----

## Interview Preparation

### Technical Questions

1. **What’s the difference between Delta Lake and Parquet?**
- Delta: ACID, time travel, MERGE, schema evolution
- Parquet: Just file format, no transactions
1. **Explain data skew and how to fix it**
- Skew: Uneven data distribution across partitions
- Fix: Salting (add random suffix to hot keys)
1. **When would you use broadcast join?**
- When one table is small (< 100MB typically)
- Avoids shuffle, much faster
1. **What is Z-ORDER and when to use it?**
- Co-locates related data using space-filling curves
- Enables data skipping
- Use for high-cardinality columns in WHERE clauses
1. **Query running slowly - what do you check?**
- Spark UI (find bottleneck stage)
- Query plan (look for shuffles)
- Data skew (check key distribution)
- Partitions (count and sizes)
- Memory (GC time)
- Delta (file count)

### Scenario-Based Questions

1. **Design a real-time streaming pipeline**
- Kafka → Bronze (raw) → Silver (cleaned) → Gold (aggregated)
- Use structured streaming with Delta
- Implement exactly-once processing
1. **How do you implement data quality checks?**
- Schema validation
- Record-level checks (nulls, ranges, formats)
- Quarantine pattern (separate valid/invalid)
- Aggregate checks (outliers, freshness)
- Monitoring and alerts
1. **Optimize a slow Spark job**
- Profile with Spark UI
- Check for data skew → Apply salting
- Use broadcast joins for small tables
- Optimize Delta tables (OPTIMIZE, Z-ORDER)
- Right-size executors and partitions

-----

## Best Practices

### Development

- Use SparkSession (not SparkContext)
- Start with Conda for environment management
- Use VS Code for Jupyter notebooks
- Enable Git integration early
- Structure projects with notebooks/, src/, data/

### Data Engineering

- Follow medallion architecture (Bronze/Silver/Gold)
- Implement data quality checks at each layer
- Use Delta Lake for all tables
- Partition by date, Z-ORDER by high-cardinality columns
- Run OPTIMIZE regularly (not after every write)

### Performance

- Default to 4-5 cores per executor
- Aim for 2-4 partitions per core
- Target 128MB-1GB per partition
- Cache only frequently reused data
- Use broadcast joins when appropriate
- Filter early (predicate pushdown)

### Production

- Use Job clusters (not All-Purpose) for scheduled jobs
- Implement quarantine pattern for bad data
- Monitor data quality metrics over time
- Set up alerts for quality threshold breaches
- Use Delta Live Tables for complex pipelines
- Enable auto-scaling for cost optimization

### Cost Optimization

- Auto-terminate idle clusters (20 min)
- Use spot instances for non-critical jobs
- Job clusters over All-Purpose
- Right-size based on actual usage
- Enable Delta cache acceleration

-----

## Quick Reference

### Essential Commands

```python
# Spark Session
spark = SparkSession.builder.appName("App").getOrCreate()

# Delta Operations
spark.sql("OPTIMIZE delta.`/path/table` ZORDER BY (col1, col2)")
spark.sql("VACUUM delta.`/path/table` RETAIN 168 HOURS")
spark.sql("DESCRIBE HISTORY delta.`/path/table`")

# Partitions
df.rdd.getNumPartitions()
df.repartition(200)
df.coalesce(50)

# Cache
df.cache()
df.persist(StorageLevel.MEMORY_AND_DISK)

# Broadcast
from pyspark.sql.functions import broadcast
large.join(broadcast(small), "key")

# Databricks Utilities
dbutils.fs.ls("/path")
dbutils.notebook.run("notebook", timeout_seconds)
dbutils.widgets.text("param", "default")
dbutils.secrets.get(scope="scope", key="key")
```

### Configuration Cheat Sheet

```python
# Performance
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.shuffle.partitions", "200")

# Delta
spark.conf.set("spark.databricks.delta.optimizeWrite.enabled", "true")
spark.conf.set("spark.databricks.delta.autoCompact.enabled", "true")

# Memory
spark.conf.set("spark.executor.memory", "16g")
spark.conf.set("spark.executor.memoryOverhead", "2g")

# Cores
spark.conf.set("spark.executor.cores", "5")
spark.conf.set("spark.executor.instances", "29")
```

-----

## Resources

### Practice Environment

- Databricks Community Edition (free)
- Docker: `jupyter/pyspark-notebook`
- Local: `pip install pyspark`

### Documentation

- Apache Spark: https://spark.apache.org/docs/latest/
- Delta Lake: https://docs.delta.io/
- Databricks: https://docs.databricks.com/

### Learning Path

1. Setup local Spark environment
1. Practice basic DataFrame operations
1. Learn Delta Lake fundamentals
1. Implement data quality checks
1. Optimize queries using Spark UI
1. Build end-to-end pipeline (Bronze/Silver/Gold)
1. Practice on Databricks Community Edition

-----

## Companion Notebook

See `spark_databricks_practice.ipynb` for hands-on code examples covering:

- Environment setup
- Spark fundamentals
- Delta Lake operations
- Performance optimization techniques
- Data quality implementations
- Real-world scenarios

-----

**Last Updated:** December 2024
**Version:** 1.0
**Maintained by:** Your AI Engineering Journey