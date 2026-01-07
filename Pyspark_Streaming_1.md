# PySpark Structured Streaming – End-to-End Example

## Core Streaming Concepts (Mental Model)

* **Offsets** → WHAT to process
* **Trigger** → WHEN to process
* **SQL / Transformations** → HOW to process
* **State** → WHAT to remember
* **Watermark** → WHAT to forget
* **Checkpoint** → WHEN a batch is considered complete

---

## Imports

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.sql.types import *
from delta.tables import DeltaTable
```

---

## Initialize Spark Session

```python
spark = SparkSession.builder \
    .appName("ClickstreamProcessing") \
    .getOrCreate()
```

---

## Checkpoint Location

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

## 2. Read Stream (OFFSETS – WHAT to Process)

```python
clickstream_df = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "your-kafka-broker:9092") \
    .option("subscribe", "clickstream-topic") \
    .option("startingOffsets", "latest") \
    .option("maxOffsetsPerTrigger", 10000) \
    .load()
```

### Parse JSON from Kafka

```python
parsed_df = clickstream_df.select(
    from_json(col("value").cast("string"), clickstream_schema).alias("data"),
    col("timestamp").alias("kafka_timestamp"),
    col("offset"),
    col("partition")
).select("data.*", "kafka_timestamp", "offset", "partition")
```

---

## 3. Watermark (WHAT to Forget)

```python
watermarked_df = parsed_df.withWatermark("timestamp", "10 minutes")
```

---

## 4. SQL / Transformations (HOW to Process)

### SQL Style

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

### DataFrame API Style

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

## 5. Stateful Operations (WHAT to Remember)

```python
session_stats = watermarked_df \
    .withWatermark("timestamp", "30 minutes") \
    .groupBy("session_id", "user_id") \
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

## 6. Write Stream (TRIGGER + CHECKPOINT)

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

### Available-Now Trigger

```python
query3 = watermarked_df.filter(col("event_type") == "purchase") \
    .writeStream \
    .format("delta") \
    .outputMode("append") \
    .trigger(availableNow=True) \
    .option("checkpointLocation", f"{checkpoint_location}/purchases") \
    .option("path", "/mnt/delta/purchases") \
    .start()
```

---

## 7. Monitoring

```python
for stream in spark.streams.active:
    print(stream.id)
    print(stream.status)
    print(stream.recentProgress)
```

```python
spark.streams.awaitAnyTermination()
```

---

## 8. Checkpoint & Recovery

Checkpoint stores:

* Processed offsets
* Stateful aggregation data
* Metadata for fault tolerance

```python
# ⚠️ Use carefully
dbutils.fs.rm(checkpoint_location, recurse=True)
```

---

## Key Takeaways

* **Offsets** → control ingestion
* **Trigger** → control execution timing
* **State** → enables session/window logic
* **Watermark** → manages late data
* **Checkpoint** → guarantees fault tolerance & exactly-once semantics

---

This structure is **GitHub-clean**, **readable**, and **production-ready**.

## 🔍 Concept Validation – Is This Statement Correct?

> **Offsets decide WHAT to process**
> ✅ **TRUE** – Offsets determine which Kafka records Spark will read next.

> **Trigger decides WHEN to process**
> ✅ **TRUE** – Trigger controls *when Spark starts a micro-batch* (time-based, once, availableNow).

> **SQL decides HOW to process**
> ✅ **TRUE** – SQL/DataFrame transformations define computation logic.

> **State decides WHAT to remember**
> ✅ **TRUE** – State stores intermediate aggregation/session data across batches.

> **Watermark decides WHAT to forget**
> ✅ **TRUE** – Watermark evicts old state and late events beyond allowed delay.

> **Checkpoint decides WHEN a batch is complete**
> ⚠️ **PARTIALLY TRUE**
> Checkpoint marks:

* What offsets are committed
* What state is persisted
  But **batch completion** is decided by **successful execution + offset commit**, not checkpoint alone.

---

## 🔄 Structured Streaming Lifecycle (End-to-End)

1. **Trigger fires**
2. Spark checks **latest offsets**
3. Creates a **micro-batch**
4. Applies **transformations**
5. Updates **state store**
6. Applies **watermark cleanup**
7. Writes to **sink**
8. Commits offsets to **checkpoint**

If **any step fails → batch is retried**.

---

## 🎯 Exactly-Once Semantics (Very Important)

Spark Structured Streaming guarantees **exactly-once** when:

* Source supports replay (Kafka)
* Sink is idempotent (Delta Lake)
* Checkpointing is enabled

### Why duplicates don’t happen:

* Offsets are committed **only after successful write**
* On restart → Spark resumes from last committed offset

---

## ⏱ How Spark Decides Micro-Batch Size

Micro-batch size depends on:

* `maxOffsetsPerTrigger`
* Trigger interval
* Source backlog
* Cluster resources

Example:

* 100k messages available
* `maxOffsetsPerTrigger = 10k`
  → Spark creates **10 micro-batches**

---

## 🧠 Watermark + State + Checkpoint (Relationship)

| Component  | Role                                    |
| ---------- | --------------------------------------- |
| State      | Stores running aggregation/session data |
| Watermark  | Cleans old state                        |
| Checkpoint | Persists state + offsets                |

💡 **Without watermark → state grows forever**

---

## ⚠️ Common Misconceptions

❌ *Checkpoint removes duplicates*
✅ Offset management + idempotent sink removes duplicates

❌ *Watermark drops data immediately*
✅ It drops data **only after watermark delay**

❌ *Trigger affects correctness*
✅ Trigger affects **latency**, not correctness

---

## 🧪 Interview One-Liners

* **Offsets**: Define read position in source
* **Trigger**: Controls execution frequency
* **Watermark**: Manages late data & state eviction
* **State**: Enables session/window aggregations
* **Checkpoint**: Ensures fault tolerance & recovery
* **Structured Streaming**: Micro-batch execution engine

---

## 📌 When to Use Which Trigger

| Trigger        | Use Case                 |
| -------------- | ------------------------ |
| processingTime | Continuous streaming     |
| once           | Batch replacement        |
| availableNow   | Backfill + stop          |
| continuous     | Ultra-low latency (rare) |

---

## 🧩 Final Mental Model

> **Structured Streaming = Deterministic replayable batch engine**

Everything is:

* Restartable
* Fault-tolerant
* Exactly-once

---

This turns your page from **“code dump” → “expert reference doc”**.

