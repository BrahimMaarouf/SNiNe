# CityPulse Lakehouse: milestones

> Format: **Goal → Why now → Concepts → What to build (Core / Stretch) → Best practices → Definition of Done → Pitfalls**. Total: 7 milestones over about 5 days.

---

## 💸 Zero-cost rule: this whole project costs $0 and needs no card

Everything runs **on my PC** (i9-14900HX, 32 GB RAM, ~1.3 TB free SSD) with free, open-source software in WSL2/Docker. No cloud account is needed.

| Need | Free choice ✅ | Avoid ❌ (needs a card or money) |
|------|---------------|----------------------------------|
| Data | NYC TLC Parquet files + zone lookup (free public download) | — |
| Compute | **Spark 4 local mode** in WSL2 or Docker | EMR, Dataproc, HDInsight, Synapse |
| Storage | Local disk, then **Garage** / **SeaweedFS** (S3-compatible) in Docker | Paid cloud buckets |
| Streaming | **Apache Kafka (KRaft)** or **Redpanda** in Docker | Confluent Cloud, MSK, Event Hubs |
| Orchestration | **Apache Airflow 3** in Docker Compose | Astronomer, MWAA, Cloud Composer |
| Quality / lineage | **GX Core** or **Soda Core**, **OpenLineage + Marquez** | GX Cloud, Soda Cloud (paid tiers) |
| Query + dashboard | **DuckDB** / **Trino**, **Superset** / **Metabase** (open source) | Hosted BI tools |
| Managed Spark (stretch only) | **Databricks Free Edition** (free sign-up with email, no card; check its current limits) | Databricks trials on AWS/Azure (they need cloud billing) |

**My hard rules:** never enter a card; if a "free trial" asks for one, skip it. Free courses and docs only (Confluent Developer courses, Apache docs, DataTalksClub Data Engineering Zoomcamp).

**Laptop tip:** this stack is heavy. I give WSL2 **12–16 GB** in `C:\Users\bdaar\.wslconfig` (`memory=14GB`, `processors=12`) and only start the services the current milestone needs. Kafka, Airflow, Superset, and Spark don't all have to run at once.

---

## M0: Big Data mindset and environment (Day 1 morning, ~3 h)

**Goal:** a working Spark + Docker environment and a clear idea of **why** distributed systems exist.

**Why now:** if I don't understand *why* data is distributed, Spark looks like "slow pandas". Setting up the environment first avoids losing Day 3 to Java errors.

**Concepts to learn**
- The 5 Vs (Volume, Velocity, Variety, Veracity, Value)
- Scale-up vs scale-out; why disks/network are the bottleneck, not the CPU
- The CAP theorem (quick intuition only)
- A short history: GFS/MapReduce papers → Hadoop → Spark → lakehouse. Read the **summary** of the original MapReduce paper.
- Data lake vs data warehouse vs lakehouse

**What to build**
- Core: a WSL2 (or Docker) environment with Java 17/21 and PySpark 4.x. Run a tiny Spark job and **open the Spark UI** (port 4040).
- Core: a Docker Compose skeleton (I'll add Kafka, Airflow, and the others later).
- Core: `docs/architecture.md` with the target diagram drawn by me.
- Core: download the **taxi zone lookup** CSV and **one** month of yellow taxi Parquet, and read the TLC **data dictionary**.

**Definition of Done**
- [ ] Spark job runs and I can find its jobs/stages in the Spark UI
- [ ] I can explain the lakehouse in 3 sentences

**Pitfalls:** running Spark natively on Windows; the Java version doesn't match the Spark version.

---

## M1: Ingestion and storage formats, building Bronze (Day 1 afternoon, ~4 h)

**Goal:** ingest several months of raw data **idempotently** and understand why storage format decisions matter more than code.

**Why now:** everything downstream depends on how data lands. Bad storage choices (CSV, no partitioning, millions of tiny files) can't be fixed by clever Spark code later.

**Concepts to learn**
- Row-oriented (CSV, JSON, Avro) vs columnar (Parquet, ORC) storage; predicate pushdown; column pruning; compression codecs (snappy, zstd)
- Parquet internals: row groups, column chunks, statistics (min/max)
- Partitioning (by year/month) vs over-partitioning; the **small files problem**
- Idempotency: running the same ingestion twice must not duplicate data
- Schema-on-read vs schema-on-write

**What to build**
- Core: an ingestion script that downloads N months (configurable, never hard-coded), checks file integrity (size/row count), and lands the files untouched in `bronze/` with ingestion metadata (ingestion timestamp, source file name).
- Core: **experiment**: convert one month to CSV, then compare CSV vs Parquet (snappy) vs Parquet (zstd) on **file size** and **time to compute an average fare** in Spark. Put the results in a table in `docs/benchmarks.md`.
- Core: re-run ingestion and prove there are no duplicates (idempotency).
- Stretch: note **schema drift** between months/years (TLC column types changed over time) and design how I handle it.

**Best practices:** raw data is immutable; record lineage metadata; configuration lives in a file or env vars; log row counts at every step.

**Definition of Done**
- [ ] ≥ 3 months in Bronze, re-runnable with no duplicates
- [ ] Format benchmark table written
- [ ] I can explain why Parquet + partitioning makes queries faster

---

## M2: Spark internals and the Silver layer (Day 2, ~6 h)

**Goal:** clean, validate, and enrich the data with Spark, and **understand what Spark does under the hood** by reading the execution plan and the Spark UI.

**Why now:** this is the heart of Big Data engineering. A real data engineer is judged on writing **correct and efficient** distributed transformations, and I can't optimise what I can't read.

**Concepts to learn**
- Driver vs executors; jobs → stages → tasks; partitions
- Transformations vs actions; lazy evaluation; the DAG
- **Narrow vs wide transformations**, and why **shuffle** is expensive
- The Catalyst optimizer, `explain()` plans (logical → physical)
- Join strategies: **broadcast hash join** vs sort-merge join
- **Adaptive Query Execution (AQE)**: coalescing partitions, skew join handling
- Caching/persistence, and when *not* to cache
- Data cleaning rules and quarantine (dead-letter) tables

**What to build**
- Core: a Silver job that:
  - enforces an explicit **schema** (no type inference in production)
  - filters invalid trips with documented business rules (negative fares, zero distance with non-zero fare, drop-off before pick-up, passenger_count outliers, trip duration > 6 h…), writing rejects to a **quarantine** table with a `reject_reason` column
  - deduplicates
  - adds derived columns (duration, speed, hour, day of week, is_weekend, tip percentage)
  - joins the pick-up/drop-off zones using a **broadcast join**
- Core: **performance experiment**: run the zone join with broadcast forced vs disabled, and with AQE on vs off. Record the time, the number of stages, and the shuffle read/write from the Spark UI, and add screenshots to `docs/benchmarks.md`.
- Core: write 3–5 **pytest** tests for my transformation functions using a small local SparkSession and handmade DataFrames.
- Stretch: create artificial **skew** (e.g. 80% of rows with one zone) and observe how AQE's skew-join handling (or manual salting) fixes it.

**Best practices:** transformation logic in pure, testable functions; no `collect()` on big data; avoid Python UDFs (use built-in functions; look up "why Python UDFs are slow" and "pandas UDF Arrow"); explicit schemas.

**Definition of Done**
- [ ] Silver table plus a quarantine table with reasons; the reject rate is documented
- [ ] I can read an `explain()` plan and point at the Exchange (shuffle) and the BroadcastHashJoin
- [ ] Benchmarks with Spark UI screenshots
- [ ] Unit tests pass

**Pitfalls:** `inferSchema` in production; too many/too few partitions (look up "spark.sql.shuffle.partitions"); calling `.count()` everywhere for debugging (each one triggers a full job).

---

## M3: Lakehouse table format and the Gold layer (Day 3, ~6 h)

**Goal:** turn plain Parquet folders into **ACID tables** with Apache Iceberg, and build business-ready Gold tables.

**Why now:** plain Parquet files have no transactions: a failed job leaves half-written data, updates mean rewriting everything, and nobody knows which version is current. Table formats solve this, and they're the foundation of every modern data platform.

**Concepts to learn**
- Iceberg architecture: catalog → metadata files → manifest lists → manifests → data files
- ACID on object storage; snapshot isolation; optimistic concurrency
- **Time travel**, rollback, **schema evolution**, **hidden partitioning**, partition evolution
- `MERGE INTO` (upserts), copy-on-write vs merge-on-read
- Table maintenance: compaction (rewrite data files), expiring snapshots, orphan file cleanup
- Dimensional modelling (Kimball): fact vs dimension tables, star schema, grain

**What to build**
- Core: rewrite Bronze/Silver as **Iceberg tables** in a catalog (start with a simple catalog).
- Core: Gold tables:
  - `dim_zone`, `dim_date` (or `dim_time`), `fact_trips` (state the **grain** explicitly)
  - `agg_zone_hourly` (trips, revenue, avg tip %, avg speed per zone and hour)
- Core: demonstrate **time travel** (query a table as of a previous snapshot), **schema evolution** (add a column without rewriting the data), and a **MERGE** that upserts late-arriving corrections.
- Core: run **compaction** after many small writes and compare file counts before and after.
- Stretch: switch to a **REST catalog** (Polaris / Lakekeeper) and read the same tables from **DuckDB or Trino**. This is the "open table format, many engines" idea in action.

**Best practices:** declare the grain of every fact table; Gold tables are modelled for **consumers** (analysts, ML), not for the convenience of whoever wrote the pipeline; schedule maintenance jobs.

**Definition of Done**
- [ ] Iceberg Bronze/Silver/Gold tables exist
- [ ] Time travel, schema evolution, and MERGE demonstrated and documented
- [ ] Star schema diagram in `docs/`
- [ ] I can explain how Iceberg makes a commit atomic on object storage

---

## M4: Real-time streaming with Kafka and Structured Streaming (Day 4, ~6 h)

**Goal:** process a live stream of trips with **event-time windows**, **watermarks**, and **fault tolerance**.

**Why now:** batch answers "what happened yesterday". Many businesses need "what is happening now". I come to streaming *after* batch because Structured Streaming reuses the same DataFrame API: a stream is an unbounded table.

**Concepts to learn**
- Kafka: brokers, topics, **partitions**, replication, producers (keys → partitions), **consumer groups**, offsets, retention, KRaft
- Delivery semantics: at-most-once, at-least-once, **exactly-once**
- Structured Streaming: micro-batches, triggers, output modes (append/update/complete), **checkpoints**
- **Event time vs processing time**, tumbling/sliding windows, **watermarks** and late data
- Lambda vs Kappa architecture
- Message formats and schema registries (Avro/Protobuf; for this project JSON is acceptable)

**What to build**
- Core: Kafka (KRaft) in Docker Compose, plus a topic `trips.raw` with **3 partitions**.
- Core: a **producer simulator** that replays one month of trips in event-time order at configurable speed (e.g. 1 hour of data per minute), keyed by pick-up zone, and that deliberately sends **5% of events late** (shuffled timestamps).
- Core: a Structured Streaming job that reads the topic, parses and validates events, and computes **trips & revenue per zone per 5-minute window** with a **10-minute watermark**, writing to an Iceberg Gold table (`stream_zone_5min`) with checkpointing.
- Core: **fault-tolerance test**: kill the streaming job mid-run, restart it, and prove there are no duplicates or gaps (thanks to checkpoint + offsets).
- Core: show what happens to events that arrive later than the watermark.
- Stretch: a simple **anomaly alert** (e.g. a zone whose demand is > 3× its usual hourly average) written to an `alerts` topic.
- Stretch: read about **Apache Flink** and write 1 page on when I'd choose it over Spark Streaming.

**Best practices:** choose message keys carefully (ordering is guaranteed only within a partition); never delete checkpoints in production; monitor consumer lag.

**Definition of Done**
- [ ] The live window table updates while the simulator runs
- [ ] Kill/restart test passes with no duplicates
- [ ] I can explain watermarks with a drawn timeline

**Pitfalls:** windowing on processing time instead of event time; reusing one checkpoint folder for two different queries.

---

## M5: Orchestration, data quality, and reliability (Day 5 morning, ~4 h)

**Goal:** make the batch pipeline **production-grade**: scheduled, retried, backfillable, validated, and observable.

**Why now:** scripts I run by hand aren't a pipeline. Real data platforms fail in silent ways (an upstream schema change, half the rows missing), so I need automation **and** guards.

**Concepts to learn**
- DAGs, tasks, operators, scheduling, retries, SLAs, **backfills**, catchup
- Airflow 3: **Assets** (data-aware scheduling), the task SDK, the new UI, how it decouples task execution from the scheduler/metadata DB
- **Idempotency** and determinism of tasks (partition overwrite by date)
- Data quality dimensions: completeness, validity, uniqueness, timeliness, consistency
- **Data contracts**; schema validation; "fail fast" vs "quarantine and alert"
- Data lineage (OpenLineage)

**What to build**
- Core: an Airflow 3 DAG `citypulse_monthly`: ingest → bronze → **quality gate** → silver → **quality gate** → gold → maintenance (compaction). Make it parameterised by month.
- Core: quality suites (GX or Soda) with ≥ 8 checks: not-null keys, value ranges, accepted values (payment_type), row-count thresholds vs the previous month, freshness, uniqueness. A failing check must **stop** the DAG (or quarantine) and send a visible alert.
- Core: run a **backfill** for 3 past months and prove the results are identical when re-run.
- Core: **break it on purpose**: inject a corrupted file and show that the quality gate catches it.
- Stretch: emit OpenLineage events and view the lineage graph in Marquez.

**Definition of Done**
- [ ] DAG runs end-to-end on schedule and manually, with retries configured
- [ ] Quality gate stops bad data (demonstrated)
- [ ] Backfill is idempotent

---

## M6: Serving, analytics, and the final report (Day 5 afternoon, ~3 h)

**Goal:** deliver **value**: fast queries, a dashboard, and a clear write-up.

**Why now:** a pipeline that nobody consumes has no value. Data engineers are measured on the trust and speed of the data their consumers get.

**What to build**
- Core: query the Gold tables from **DuckDB or Trino**.
- Core: a dashboard (Superset/Metabase) with: trips per hour heatmap (hour × weekday), top 10 zones by revenue, average tip % by payment type, airport vs non-airport trips, the live 5-minute stream table, and the data-quality pass rate over time.
- Core: **single-node vs Spark comparison**: run the same Gold aggregation in DuckDB or Polars on my laptop and compare with Spark. Write an honest conclusion: *when* do I actually need Spark? (Search "DuckDB vs Spark" and "the end of big data" discussions.)
- Core: final README: architecture, how to run, data dictionary, benchmark results, limitations, and what I'd change at 100× the scale.
- Stretch: run the Silver job on **Databricks Free Edition** (free, no card) on a sample of the data, and compare it with my local Spark (setup, speed, developer experience).

**Definition of Done**
- [ ] Dashboard works on Gold data
- [ ] Spark vs single-node benchmark and conclusion written
- [ ] README complete with a diagram

---

## Self-check: interview questions

1. Why is Parquet faster than CSV for analytical queries? Name 3 mechanisms.
2. What is a shuffle, which operations cause one, and how do I reduce it?
3. When does Spark choose a broadcast join? What happens if the "small" table is actually large?
4. What does AQE do at runtime that the static optimizer can't?
5. What is the small files problem, and how do Iceberg/Delta help?
6. How does Iceberg achieve atomic commits on object storage that has no transactions?
7. Explain event time, processing time, windows, and watermarks with an example from my project.
8. How did my streaming job recover with no duplicates after I killed it?
9. What makes an Airflow task idempotent? Why does it matter for backfills?
10. My dashboard shows revenue down 40% today. How do I find out whether it's real or a data bug?
11. When would I **not** use Spark?

## References to search
- "Designing Data-Intensive Applications" (Martin Kleppmann). Chapters on storage, batch, and stream processing. The **2nd edition** has been coming out recently; check which one is available.
- "Spark: The Definitive Guide" plus the official **Spark SQL performance tuning** docs
- Apache Iceberg docs: "Spec", "Maintenance", "Spark quickstart"
- Confluent Developer: "Apache Kafka 101" (free course)
- Airflow 3 docs: "Assets", "Best practices"
- "Fundamentals of Data Engineering" (Joe Reis & Matt Housley)
