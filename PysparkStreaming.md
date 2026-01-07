# PysparkStreaming

How true is this statement, Offsets decide WHAT to process
Trigger decides WHEN to process
SQL decides HOW to process
State decides WHAT to remember
Watermark decides WHAT to forget
Checkpoint decides WHEN a batch is complete

# PySpark Structured Streaming – Conceptual & Practical Guide

## Core Streaming Principles (Mental Model)

| Concept                   | Decides                                   |
| ------------------------- | ----------------------------------------- |
| **Offsets**               | WHAT to process                           |
| **Trigger**               | WHEN to process                           |
| **SQL / Transformations** | HOW to process                            |
| **State**                 | WHAT to remember                          |
| **Watermark**             | WHAT to forget                            |
| **Checkpoint**            | WHEN a batch is complete & fault-tolerant |

---

## Spark Session Initialization

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.sql.types import *
from delta.tables import DeltaTable

spark = SparkSession.builder \
    .appName("ClickstreamProcessing") \
    .getOrCreate()
```

### Checkpoint Location

```python
checkpoint_location = "/mnt/checkpoints/clickstream_app"
```

---

## 1. Define Schema (Structured Processing)

```python
clickstream_schema = StructType([
    StructField("user_id", StringType(), True),
    StructField("session_id", StringType(), True),
    StructField("event_type", StringType(), True),
    StructField("page_url", StringType(), True),
    StructField("product_id", StringType(), True),
    StructField("timestamp", TimestampType(), True),
    StructField("device_type", StringType(), True),
    StructField("ip_address", StringType(), True)
])
```

---

## 2. Read Stream – Offsets (WHAT to process)

```python
clickstream_df = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "your-kafka-broker:9092") \
    .option("subscribe", "clickstream-topic") \
    .option("startingOffsets", "latest") \
    .option("maxOffsetsPerTrigger", 10000) \
    .load()
```

### Parse Kafka JSON Payload

```python
parsed_df = clickstream_df.select(
    from_json(col("value").cast("string"), clickstream_schema).alias("data"),
    col("timestamp").alias("kafka_timestamp"),
    col("offset"),
    col("partition")
).select("data.*", "kafka_timestamp", "offset", "partition")
```

---

## 3. Watermark – WHAT to forget (Late Data Handling)

```python
watermarked_df = parsed_df \
    .withWatermark("timestamp", "10 minutes")
```

---

## 4. Transformations – HOW to process

### SQL-based Processing

```python
watermarked_df.createOrReplaceTempView("clickstream_events")

processed_df = spark.sql("""
SELECT
    user_id,
    session_id,
    event_type,
    page_url,
    product_id,
    timestamp,
    device_type,
    window(timestamp, '5 minutes') AS time_window,
    COUNT(*) AS event_count
FROM clickstream_events
WHERE event_type IN ('click', 'page_view', 'purchase')
GROUP BY
    user_id,
    session_id,
    event_type,
    page_url,
    product_id,
    timestamp,
    device_type,
    window(timestamp, '5 minutes')
""")
```

### DataFrame API Alternative

```python
user_activity_df = watermarked_df.groupBy(
    col("user_id"),
    col("session_id"),
    window(col("timestamp"), "5 minutes")
).agg(
    count("*").alias("total_events"),
    countDistinct("page_url").alias("unique_pages"),
    sum(when(col("event_type") == "click", 1).otherwise(0)).alias("click_count"),
    sum(when(col("event_type") == "purchase", 1).otherwise(0)).alias("purchase_count"),
    first("device_type").alias("device_type")
)
```

---

## 5. Stateful Operations – WHAT to remember

```python
session_stats = watermarked_df \
    .withWatermark("timestamp", "30 minutes") \
    .groupBy(col("session_id"), col("user_id")) \
    .agg(
        min("timestamp").alias("session_start"),
        max("timestamp").alias("session_end"),
        count("*").alias("events_in_session"),
        collect_list("event_type").alias("event_sequence"),
        sum(when(col("event_type") == "purchase", 1).otherwise(0)).alias("purchases")
    ) \
    .withColumn(
        "session_duration_minutes",
        (unix_timestamp("session_end") - unix_timestamp("session_start")) / 60
    )
```

---

## 6. Write Stream – Trigger & Checkpoint

### Processing Time Trigger

```python
query1 = user_activity_df.writeStream \
    .format("delta") \
    .outputMode("append") \
    .trigger(processingTime="2 minutes") \
    .option("checkpointLocation", f"{checkpoint_location}/user_activity") \
    .option("path", "/mnt/delta/user_activity") \
    .start()
```

### Once Trigger

```python
query2 = session_stats.writeStream \
    .format("delta") \
    .outputMode("update") \
    .trigger(once=True) \
    .option("checkpointLocation", f"{checkpoint_location}/session_stats") \
    .option("path", "/mnt/delta/session_stats") \
    .start()
```

### Continuous Trigger (Experimental)

```python
query3 = processed_df.writeStream \
    .format("delta") \
    .outputMode("append") \
    .trigger(continuous="1 second") \
    .option("checkpointLocation", f"{checkpoint_location}/processed_events") \
    .option("path", "/mnt/delta/processed_events") \
    .start()
```

### Available Now Trigger

```python
query4 = watermarked_df \
    .filter(col("event_type") == "purchase") \
    .writeStream \
    .format("delta") \
    .outputMode("append") \
    .trigger(availableNow=True) \
    .option("checkpointLocation", f"{checkpoint_location}/purchases") \
    .option("path", "/mnt/delta/purchases") \
    .start()
```

---

## 7. Multiple Output Sinks

### Console (Debugging)

```python
console_query = user_activity_df.writeStream \
    .format("console") \
    .outputMode("complete") \
    .trigger(processingTime="30 seconds") \
    .option("truncate", "false") \
    .start()
```

### Custom Alerts (foreachBatch)

```python
def process_alerts(batch_df, batch_id):
    alerts = batch_df.filter(
        (col("purchase_count") > 5) |
        (col("total_events") > 100)
    )

    if alerts.count() > 0:
        alerts.write.format("delta").mode("append").save("/mnt/delta/alerts")
```

---

## 8. Monitoring & Management

```python
for stream in spark.streams.active:
    print(f"Stream ID: {stream.id}")
    print(f"Status: {stream.status}")
    print(f"Recent Progress: {stream.recentProgress}")
```

```python
spark.streams.awaitAnyTermination()
```

---

## 9. Recovery & Fault Tolerance (Checkpoint)

**Checkpoint stores:**

* Offsets (WHAT processed)
* State (WHAT remembered)
* Metadata (batch progress)

### Reset (Use with Caution)

```python
dbutils.fs.rm(checkpoint_location, recurse=True)
```

---

## 10. Complete End-to-End Pipeline Example

```python
def build_clickstream_pipeline():
    raw_stream = spark.readStream \
        .format("kafka") \
        .option("kafka.bootstrap.servers", "broker:9092") \
        .option("subscribe", "clicks") \
        .option("startingOffsets", "latest") \
        .load()

    parsed = raw_stream \
        .select(from_json(col("value").cast("string"), clickstream_schema).alias("data")) \
        .select("data.*") \
        .withWatermark("timestamp", "15 minutes")

    aggregated = parsed.groupBy(
        window(col("timestamp"), "5 minutes", "1 minute"),
        col("user_id")
    ).agg(
        count("*").alias("event_count"),
        countDistinct("session_id").alias("session_count")
    )

    return aggregated.writeStream \
        .format("delta") \
        .outputMode("append") \
        .trigger(processingTime="2 minutes") \
        .option("checkpointLocation", f"{checkpoint_location}/main_pipeline") \
        .option("path", "/mnt/delta/clickstream_aggregated") \
        .start()
```

---

## Key Takeaways

* **Offsets** → What data is consumed
* **Trigger** → When Spark runs a micro-batch
* **SQL / DataFrame API** → How data is processed
* **State** → What Spark remembers between batches
* **Watermark** → How late data is handled
* **Checkpoint** → Fault tolerance & exactly-once semantics


