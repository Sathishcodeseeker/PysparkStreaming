# Watermark vs Checkpoint in Spark Structured Streaming

## Key Difference

**Watermark ≠ Checkpoint**

| Concept        | What it is                    | Purpose                                    |
| -------------- | ----------------------------- | ------------------------------------------ |
| **Watermark**  | Event-time progress indicator | Handles late data & closes windows         |
| **Checkpoint** | Fault-tolerance mechanism     | Enables recovery & exactly-once processing |

---

## What is a Watermark?

A **watermark** tells Spark:

> *“I’m confident that I’ve seen all events up to time **T** in **event time**.”*

```python
df.withWatermark("timestamp", "10 minutes")
```

This means:

> *Wait up to **10 minutes** for late events.
> After that, older events are considered **too late** and will be dropped.*

---

## How Watermark Is Calculated

```
Current Watermark = Max Event Time Seen So Far - Watermark Delay
```

### Example

```python
clickstream_df.withWatermark("timestamp", "10 minutes")
```

**Batch 1 events**

| Event | Event Time |
| ----- | ---------- |
| A     | 10:00      |
| B     | 10:05      |
| C     | 10:03      |

```
Max event time = 10:05
Watermark = 10:05 - 10 minutes = 09:55
```

✔ Events with timestamp **≥ 09:55** → Accepted
❌ Events **< 09:55** → Dropped as late

---

## Visual Timeline (Event Time)

```
09:50   09:55   10:00   10:05   10:10
|-------|-------|-------|-------|
```

**Processing Batch 1 (real time: 10:20)**

* Events received: `10:00, 10:05, 10:03, 09:57`
* Watermark = `09:55`
* `09:57` → ✅ Accepted
* `09:50` → ❌ Dropped

**Processing Batch 2**

* New max event time = `10:15`
* New watermark = `10:05`
* Events `< 10:05` → Dropped

---

## What is a Checkpoint?

A **checkpoint** is a **persistent storage location** used to store:

* Kafka offsets
* Stateful aggregation data
* Metadata (batch IDs, configs)

```python
query = df.writeStream \
    .option("checkpointLocation", "/mnt/checkpoint/my_stream") \
    .start()
```

---

## Checkpoint Directory Structure

```
/mnt/checkpoint/my_stream/
├── commits/     # Completed batches
├── offsets/     # Kafka offsets
├── state/       # Aggregation & window state
└── metadata     # Stream configuration
```

---

## How Checkpointing Works

```text
10:00  Batch 0 → offsets 0–1000 → checkpointed
10:02  Batch 1 → offsets 1001–2000 → checkpointed
10:03  💥 Crash
10:05  Restart → resume from offset 2001
```

✔ No data loss
✔ No duplicates
✔ Exactly-once processing

---

## Side-by-Side Comparison

| Aspect      | Watermark           | Checkpoint          |
| ----------- | ------------------- | ------------------- |
| Tracks      | Event-time progress | Processing progress |
| Time domain | Event time          | Processing time     |
| Purpose     | Late data handling  | Fault tolerance     |
| Storage     | In-memory           | Persistent          |
| Decides     | Drop/accept events  | Resume point        |
| Example     | `09:55`             | Kafka offset `2000` |

---

## How Watermark and Checkpoint Work Together

```python
query = spark.readStream \
    .format("kafka") \
    .option("subscribe", "clicks") \
    .load() \
    .selectExpr("CAST(value AS STRING)") \
    .withWatermark("event_timestamp", "15 minutes") \
    .groupBy(
        window("event_timestamp", "5 minutes"),
        "user_id"
    ) \
    .agg(count("*").alias("event_count")) \
    .writeStream \
    .format("delta") \
    .outputMode("append") \
    .option("checkpointLocation", "/checkpoint/clicks") \
    .trigger(processingTime="2 minutes") \
    .start()
```

* **Watermark** → Decides which events are late
* **Checkpoint** → Ensures recovery & exactly-once writes

---

## Common Misconceptions

❌ *Watermark is last checkpoint time*
✅ Watermark is based on **event timestamps**

❌ *Checkpoint depends on watermark*
✅ Checkpoint happens **every batch**

❌ *Watermark tracks arrival time*
✅ Watermark tracks **event time only**

❌ *Watermark = current time − delay*
✅ Watermark = **max event time − delay**

---

## How Does Spark Know an Event Is Late?

**It does NOT track arrival time.**

Spark only checks:

1. **Maximum event timestamp seen so far**
2. **Is this event’s timestamp ≥ watermark?**

### Example

```text
Max event time seen = 10:05
Watermark = 09:55

Event timestamp = 09:45 → ❌ Dropped
Event timestamp = 10:00 → ✅ Accepted
```

Even if the 09:45 event arrives *now*, it is still dropped.

---

## Mental Model

* **Watermark:**
  *“Am I willing to wait for this event based on its timestamp?”*

* **Checkpoint:**
  *“Where do I resume if the job crashes?”*

---

## Final Summary

✔ Watermark controls **late data handling**
✔ Checkpoint ensures **fault tolerance**
✔ Both are required for **correct streaming pipelines**

---
