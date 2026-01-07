# PysparkStreaming

How true is this statement, Offsets decide WHAT to process
Trigger decides WHEN to process
SQL decides HOW to process
State decides WHAT to remember
Watermark decides WHAT to forget
Checkpoint decides WHEN a batch is complete

from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.sql.types import *
from delta.tables import DeltaTable

# Initialize Spark Session (usually already available in Databricks as 'spark')
spark = SparkSession.builder \
    .appName("ClickstreamProcessing") \
    .getOrCreate()

# Set checkpoint location
checkpoint_location = "/mnt/checkpoints/clickstream_app"

# ============================================================================
# 1. DEFINE SCHEMA (for structured processing)
# ============================================================================
clickstream_schema = StructType([
    StructField("user_id", StringType(), True),
    StructField("session_id", StringType(), True),
    StructField("event_type", StringType(), True),  # click, page_view, purchase
    StructField("page_url", StringType(), True),
    StructField("product_id", StringType(), True),
    StructField("timestamp", TimestampType(), True),
    StructField("device_type", StringType(), True),
    StructField("ip_address", StringType(), True)
])

# ============================================================================
# 2. READ STREAM (OFFSETS - WHAT to process)
# ============================================================================
# Reading from Kafka - offsets determine which records to consume
clickstream_df = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "your-kafka-broker:9092") \
    .option("subscribe", "clickstream-topic") \
    .option("startingOffsets", "latest") \
    .option("maxOffsetsPerTrigger", 10000) \
    .load()

# Parse JSON data from Kafka value
parsed_df = clickstream_df \
    .select(
        from_json(col("value").cast("string"), clickstream_schema).alias("data"),
        col("timestamp").alias("kafka_timestamp"),
        col("offset"),
        col("partition")
    ) \
    .select("data.*", "kafka_timestamp", "offset", "partition")

# ============================================================================
# 3. WATERMARK (WHAT to forget - handle late data)
# ============================================================================
# Allow data that's up to 10 minutes late
watermarked_df = parsed_df \
    .withWatermark("timestamp", "10 minutes")

# ============================================================================
# 4. SQL/TRANSFORMATIONS (HOW to process)
# ============================================================================

# Create temp view for SQL processing
watermarked_df.createOrReplaceTempView("clickstream_events")

# SQL-based transformations
processed_df = spark.sql("""
    SELECT 
        user_id,
        session_id,
        event_type,
        page_url,
        product_id,
        timestamp,
        device_type,
        window(timestamp, '5 minutes') as time_window,
        COUNT(*) as event_count
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

# Alternative: DataFrame API transformations (HOW to process)
# Aggregations with windowing
user_activity_df = watermarked_df \
    .groupBy(
        col("user_id"),
        col("session_id"),
        window(col("timestamp"), "5 minutes")
    ) \
    .agg(
        count("*").alias("total_events"),
        countDistinct("page_url").alias("unique_pages"),
        sum(when(col("event_type") == "click", 1).otherwise(0)).alias("click_count"),
        sum(when(col("event_type") == "purchase", 1).otherwise(0)).alias("purchase_count"),
        first("device_type").alias("device_type")
    )

# ============================================================================
# 5. STATEFUL OPERATIONS (WHAT to remember)
# ============================================================================

# Session-based aggregation (maintains state per session)
session_stats = watermarked_df \
    .withWatermark("timestamp", "30 minutes") \
    .groupBy(
        col("session_id"),
        col("user_id")
    ) \
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

# Detect user behavior patterns (stateful)
from pyspark.sql.streaming import GroupStateTimeout

def update_user_state(key, events, state):
    """
    Custom stateful function to track user behavior across batches
    STATE decides WHAT to remember
    """
    # Implementation of custom state logic
    pass

# ============================================================================
# 6. WRITE STREAM with TRIGGER and CHECKPOINT
# ============================================================================

# TRIGGER - WHEN to process (multiple options)

# Option 1: Processing time trigger (every 2 minutes)
query1 = user_activity_df.writeStream \
    .format("delta") \
    .outputMode("append") \
    .trigger(processingTime="2 minutes") \
    .option("checkpointLocation", f"{checkpoint_location}/user_activity") \
    .option("path", "/mnt/delta/user_activity") \
    .start()

# Option 2: Once trigger (process available data once and stop)
query2 = session_stats.writeStream \
    .format("delta") \
    .outputMode("update") \
    .trigger(once=True) \
    .option("checkpointLocation", f"{checkpoint_location}/session_stats") \
    .option("path", "/mnt/delta/session_stats") \
    .start()

# Option 3: Continuous trigger (low latency, experimental)
query3 = processed_df.writeStream \
    .format("delta") \
    .outputMode("append") \
    .trigger(continuous="1 second") \
    .option("checkpointLocation", f"{checkpoint_location}/processed_events") \
    .option("path", "/mnt/delta/processed_events") \
    .start()

# Option 4: Available now trigger (process all available data then stop)
query4 = watermarked_df \
    .filter(col("event_type") == "purchase") \
    .writeStream \
    .format("delta") \
    .outputMode("append") \
    .trigger(availableNow=True) \
    .option("checkpointLocation", f"{checkpoint_location}/purchases") \
    .option("path", "/mnt/delta/purchases") \
    .start()

# ============================================================================
# 7. MULTIPLE OUTPUT SINKS
# ============================================================================

# Real-time dashboard (console for debugging)
console_query = user_activity_df.writeStream \
    .format("console") \
    .outputMode("complete") \
    .trigger(processingTime="30 seconds") \
    .option("truncate", "false") \
    .start()

# Alert system (foreach batch for custom processing)
def process_alerts(batch_df, batch_id):
    """
    Custom processing for each micro-batch
    CHECKPOINT ensures fault tolerance across batches
    """
    # Filter high-value events
    alerts = batch_df.filter(
        (col("purchase_count") > 5) | 
        (col("total_events") > 100)
    )
    
    if alerts.count() > 0:
        # Send to alert system, write to database, etc.
        alerts.write.format("delta").mode("append").save("/mnt/delta/alerts")

alert_query = user_activity_df.writeStream \
    .foreachBatch(process_alerts) \
    .trigger(processingTime="1 minute") \
    .option("checkpointLocation", f"{checkpoint_location}/alerts") \
    .start()

# ============================================================================
# 8. MONITORING AND MANAGEMENT
# ============================================================================

# Monitor active streams
active_streams = spark.streams.active
for stream in active_streams:
    print(f"Stream ID: {stream.id}")
    print(f"Status: {stream.status}")
    print(f"Recent Progress: {stream.recentProgress}")

# Await termination (keep notebook running)
# query1.awaitTermination()

# Or use this for multiple queries
spark.streams.awaitAnyTermination()

# ============================================================================
# 9. RECOVERY AND FAULT TOLERANCE (CHECKPOINT)
# ============================================================================

# If stream fails, restart from checkpoint
# The checkpoint location contains:
# - Offsets processed (WHAT was processed)
# - State information (WHAT to remember)
# - Metadata about the stream

# To reset and start fresh (use with caution):
# dbutils.fs.rm(checkpoint_location, recurse=True)

# ============================================================================
# 10. EXAMPLE: COMPLETE PIPELINE
# ============================================================================

def build_clickstream_pipeline():
    """
    Complete pipeline demonstrating all concepts
    """
    # Read (OFFSETS)
    raw_stream = spark.readStream \
        .format("kafka") \
        .option("kafka.bootstrap.servers", "broker:9092") \
        .option("subscribe", "clicks") \
        .option("startingOffsets", "latest") \
        .load()
    
    # Parse and watermark (WATERMARK)
    parsed = raw_stream \
        .select(from_json(col("value").cast("string"), clickstream_schema).alias("data")) \
        .select("data.*") \
        .withWatermark("timestamp", "15 minutes")
    
    # Transform (SQL/HOW)
    aggregated = parsed \
        .groupBy(
            window(col("timestamp"), "5 minutes", "1 minute"),
            col("user_id")
        ) \
        .agg(
            count("*").alias("event_count"),
            countDistinct("session_id").alias("session_count")
        )  # STATE maintained for aggregations
    
    # Write (TRIGGER + CHECKPOINT)
    return aggregated.writeStream \
        .format("delta") \
        .outputMode("append") \
        .trigger(processingTime="2 minutes") \
        .option("checkpointLocation", f"{checkpoint_location}/main_pipeline") \
        .option("path", "/mnt/delta/clickstream_aggregated") \
        .start()

# Start the pipeline
# main_query = build_clickstream_pipeline()

Key Concepts Highlighted:

Offsets: Controlled via startingOffsets, maxOffsetsPerTrigger
Trigger: Multiple examples (processingTime, once, continuous, availableNow)
SQL: Both SQL queries and DataFrame API shown
State: Implicit in aggregations, explicit in custom stateful functions
Watermark: withWatermark() for late data handling
Checkpoint: checkpointLocation for fault tolerance and exactly-once semantics

This skeleton can be adapted based on your specific Kafka configuration, Delta Lake paths, and business logic requirements!

