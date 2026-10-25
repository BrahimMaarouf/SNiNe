# CityPulse Lakehouse: batch + streaming lakehouse on NYC taxi data

**Field:** Big Data Engineering · **Duration:** 5 days (~28 h) · **Level at the end:** junior data engineer who understands distributed processing, not just pandas

---

## 1. The story (project idea)

A city mobility agency wants to understand and **react in near-real-time** to taxi demand: busiest zones, revenue per hour, anomalous trips, and rush-hour patterns. They hand me **tens of millions of trip records** (too big to be comfortable in pandas) plus a live feed of new trips.

I build a **lakehouse** using the **Medallion architecture**:

- **Bronze:** raw data ingested as-is (batch files + Kafka stream)
- **Silver:** cleaned, validated, joined, deduplicated
- **Gold:** business-ready aggregates and a star schema for analysts and ML

I run everything with **Apache Spark**, store it in an **open table format (Apache Iceberg)**, orchestrate it with **Apache Airflow**, guard it with **data-quality checks**, and serve it to a dashboard through a fast SQL engine.

This is, almost one for one, what data engineering teams at banks, telcos, and e-commerce companies build in 2026.

---

## 2. Why this project covers the field

| Course concept | Where I practise it |
|----------------|------------------------|
| The 5 Vs of Big Data, scale-up vs scale-out, the CAP theorem | M0 |
| HDFS ideas → object storage; row vs columnar formats (CSV, JSON, Avro, Parquet, ORC); compression; partitioning | M1 |
| MapReduce → Spark: RDD/DataFrame, lazy evaluation, DAG, narrow vs wide transformations, shuffle, partitions, the Catalyst optimizer, AQE | M2 |
| Data lake vs data warehouse vs **lakehouse**; ACID on files; table formats (Iceberg / Delta / Hudi); dimensional modelling | M3 |
| Stream processing: Kafka (topics, partitions, consumer groups, offsets), Structured Streaming, windows, watermarks, exactly-once, Lambda vs Kappa | M4 |
| Orchestration, idempotency, backfills, data quality, data contracts, lineage | M5 |
| Serving layer, MPP/OLAP query engines | M6 |

---

## 3. Dataset

- **NYC TLC Trip Record Data** (official, free, monthly **Parquet** files for yellow/green taxis and FHV/Uber-Lyft trips) plus the **Taxi Zone Lookup** CSV.
  Search: "NYC TLC trip record data". Start with **3 months of yellow taxi data** (~10M+ rows) and scale to 6–12 months, or add the FHVHV files, once things work.
- Why this dataset: it's real, messy (negative fares, zero distances, impossible timestamps), large enough to need Spark, and it's an industry-standard benchmark, so I can compare my results with others.

---

## 4. Stack (current as of October 2026)

| Layer | Tool (preferred) | Alternatives | Note |
|-------|------------------|--------------|------|
| Compute | **Apache Spark 4.x (PySpark)**. 4.1.x is the current stable line; check the release page | Polars / DuckDB for comparison | Requires Java 17 or 21 |
| Where to run Spark | **WSL2 Ubuntu or a Docker container** | Native Windows (painful: winutils/Hadoop issues) | **Strongly avoid native Windows Spark** |
| Table format | **Apache Iceberg** | Delta Lake | Iceberg is the most widely adopted *open* format across engines (Spark, Trino, DuckDB, Snowflake, BigQuery…) |
| Catalog | Iceberg **REST catalog** (e.g. Apache Polaris, Lakekeeper) or a simple JDBC/SQL catalog to start | Hadoop catalog (local only), Nessie | Start simple, upgrade as a stretch goal |
| Storage | Local filesystem first, then an S3-compatible store | **Garage** / **SeaweedFS** in Docker (free, open source) | Note: MinIO's community edition changed its distribution model in 2025, so check its current status before I choose it |
| Streaming | **Apache Kafka 4.x (KRaft mode, no ZooKeeper)** | Redpanda (Kafka-compatible, lighter) | ZooKeeper was removed in Kafka 4.0 |
| Stream processing | **Spark Structured Streaming** | Apache Flink (stretch reading) | — |
| Orchestration | **Apache Airflow 3.x** (3.3.x was the latest line in mid/late 2026) | Dagster, Prefect | Airflow 3 introduced **Assets** (data-aware scheduling) and a new UI |
| Data quality | **Great Expectations (GX Core)** or **Soda Core** | pandera, dbt tests | — |
| Query / serving | **DuckDB** (reads Iceberg) or **Trino** | ClickHouse | — |
| Dashboard | **Apache Superset** or **Metabase** | Streamlit | — |
| Lineage (stretch) | **OpenLineage + Marquez** | — | — |
| Everything runs in | **Docker Compose** (from project 01) | — | — |

---

## 5. Target architecture

```
  NYC TLC Parquet (monthly)          Trip "live feed" simulator
          │ batch                              │ producer (replays a month at Nx speed)
          ▼                                    ▼
   ┌──────────────┐                    ┌──────────────┐
   │ BRONZE       │◀── Structured ─────│ Kafka topic  │
   │ raw Iceberg  │    Streaming       │ trips.raw    │
   └──────┬───────┘                    └──────────────┘
          │ Spark batch: clean, validate, dedupe, join zones (broadcast)
          ▼
   ┌──────────────┐   Great Expectations / Soda checks (fail ⇒ quarantine table)
   │ SILVER       │
   └──────┬───────┘
          │ Spark: star schema + aggregates
          ▼
   ┌──────────────┐        ┌──────────┐      ┌───────────────────┐
   │ GOLD         │──────▶ │ DuckDB / │────▶ │ Superset/Metabase │
   │ fact_trips,  │        │ Trino    │      │ dashboard         │
   │ dim_zone,    │        └──────────┘      └───────────────────┘
   │ agg_hourly   │
   └──────────────┘
        ▲ all jobs scheduled, retried and backfilled by Airflow 3
```

---

## 6. Final deliverables
1. Repo `citypulse/` with `ingestion/`, `spark_jobs/`, `streaming/`, `dags/`, `quality/`, `docker/`, `docs/` (architecture + ADRs + data dictionary).
2. Bronze/silver/gold Iceberg tables I can query, with **time travel** demonstrated.
3. A live streaming job that updates a "trips per zone per 5 min" table.
4. An Airflow DAG that rebuilds a month **idempotently** and supports **backfill**.
5. A **performance report**: CSV vs Parquet, partitioned vs not, broadcast join vs shuffle join, AQE on vs off, Spark vs DuckDB/Polars on a single machine. All numbers measured by me.
6. A dashboard with 4–6 business charts.

## 7. If I skipped project 01
I only need Docker + Docker Compose and basic Linux/WSL2 skills. Spend 2 hours on "Docker Compose getting started" first.

## 8. An honest note: "Big Data on a laptop"
I'll run Spark in **local mode** (one machine, many cores). The **concepts** (partitions, shuffles, stages, executors) are exactly the same on a 100-node cluster; only the scale changes. As a stretch goal, run my job on **Databricks Free Edition** (free, no card). The whole project costs **$0**: see the zero-cost rule at the top of `01_MILESTONES.md`.
