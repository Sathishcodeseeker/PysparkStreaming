# Micro-Batch Execution Model (Spark Structured Streaming)

In **Spark Structured Streaming**, **each trigger produces exactly one atomic micro-batch**.
That micro-batch is processed **end-to-end** before the next trigger is allowed to start.

Each micro-batch goes through **four strictly sequential stages**.

---

## High-Level Flow

```text
TRIGGER FIRES
     ↓
┌──────────────────────────────────────────┐
│        MICRO-BATCH (Atomic Unit)         │
├──────────────────────────────────────────┤
│ 1. READ        → Fetch new data          │
│ 2. EXECUTE     → Apply transformations  │
│ 3. WRITE       → Persist results         │
│ 4. CHECKPOINT  → Commit state & offsets │
└──────────────────────────────────────────┘
     ↓
NEXT TRIGGER (waits for completion)
```

---

## Stage 1: READ (I/O Bound)

The **READ** stage is responsible for fetching new data from the source.

Key points:

* Happens **once per trigger**
* Determines **batch size**
* Lazy until execution begins

### Example: Reading from Kafka

```python
clickstream_df = (
    spark.readStream
         .format("kafka")
         .option("kafka.bootstrap.servers", "broker:9092")
         .option("subscribe", "clicks")
         .option("maxOffsetsPerTrigger", 10000)  # Controls micro-batch size
         .load()
)
```

**Notes:**

* `maxOffsetsPerTrigger` limits how much data is pulled per micro-batch
* Spark tracks offsets internally
* No computation happens yet (lazy evaluation)

---

## Key Characteristics of Micro-Batches

* **Atomic**: All stages succeed or the batch is retried
* **Sequential**: No overlap between micro-batches
* **Deterministic**: Same input → same output
* **Fault-tolerant**: Recovery is driven by checkpoints

---

## Summary

* One trigger = **one micro-batch**
* Micro-batches are processed **serially**
* Each micro-batch follows:
  **READ → EXECUTE → WRITE → CHECKPOINT**
* Next trigger **waits** until the current batch fully completes

---

This execution model is what gives Structured Streaming its:

* Exactly-once guarantees
* Strong consistency
* Predictable behavior

---

## Stage 2: EXECUTE (CPU / Memory Bound)

This is where **actual computation happens**.
All transformations defined earlier are now **materialized**.

### Key Characteristics

* Triggered **only when a write action starts**
* Uses **CPU and memory heavily**
* Can be **stateless or stateful**
* May spill to disk if memory is insufficient

### What Happens Internally

* Spark creates a **physical execution plan**
* Tasks are distributed across executors
* Operations include:

  * JSON deserialization
  * Filtering and projections
  * Watermark evaluation
  * Stateful aggregations (stored in state store)

### Example (Stateful Execution)

```python
processed_df = (
    clickstream_df
        .withWatermark("timestamp", "10 minutes")
        .groupBy(
            window(col("timestamp"), "5 minutes"),
            col("user_id")
        )
        .agg(
            count("*").alias("event_count"),
            sum("purchase_amount").alias("total_spend")
        )
)
```

**Important Notes:**

* Watermarks decide **when old state can be dropped**
* Late data beyond watermark is **silently ignored**
* State is kept **per key + window**

---

## Stage 3: WRITE (I/O Bound — Often the Bottleneck)

The **WRITE** stage persists results to the sink.

### Characteristics

* Strongly **I/O bound**
* Often the **slowest stage**
* Exactly-once guarantees depend heavily on sink behavior

### Example: Writing to Delta Lake

```python
query = (
    processed_df.writeStream
        .format("delta")
        .outputMode("append")
        .option("checkpointLocation", "/chk/clickstream")
        .start("/data/clickstream_agg")
)
```

### What Happens

* Spark writes **only the output of this micro-batch**
* Sink commits must be **atomic**
* If write fails:

  * Entire micro-batch is retried
  * No partial data is exposed

### Common Sinks

* Delta Lake
* Kafka
* Parquet / ORC
* Console (debug only)

---

## Stage 4: CHECKPOINT (State + Offset Commit)

Checkpointing is what makes **Structured Streaming fault-tolerant**.

### What Gets Checkpointed

* Source offsets (Kafka offsets, file offsets)
* State store snapshots
* Metadata about completed micro-batches

### Where It Happens

* After a **successful write**
* Stored in a **reliable storage** (HDFS, ADLS, S3)

### Example Directory Structure

```text
/chk/clickstream/
├── offsets/
├── commits/
├── state/
└── sources/
```

### Failure & Recovery Scenario

1. Micro-batch completes WRITE
2. Checkpoint is committed
3. Spark crashes
4. On restart:

   * Reads last committed offsets
   * Restores state
   * Resumes exactly where it stopped

---

## End-to-End Guarantees

* **Exactly-once processing** (with supported sinks)
* **No duplicate data**
* **No data loss**
* Deterministic replay after failure

---

## Final Summary

| Stage      | Purpose                 | Resource Type |
| ---------- | ----------------------- | ------------- |
| READ       | Fetch new data          | I/O           |
| EXECUTE    | Transform & aggregate   | CPU / Memory  |
| WRITE      | Persist results         | I/O           |
| CHECKPOINT | Commit progress & state | Disk / I/O    |

Each micro-batch must **fully complete all four stages**
before the **next trigger** is allowed to run.

