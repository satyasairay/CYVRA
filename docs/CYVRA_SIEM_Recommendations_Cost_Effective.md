# CYVRA SIEM — Cost-Effective Recommendations (2025 → 2040) V1

This document focuses on concrete, actionable choices to build the most cost-effective SIEM possible while preserving detection quality, explainability, and operational simplicity. It is intentionally prescriptive — minimal fluff, specific components, trade-offs, and operational rules.

## High-level thesis
- Prioritize structured, source-side filtering and enrichment to reduce data volume entering the system.
- Use columnar storage for searchable fields and object storage for cold or raw data.
- Index only fields that are queried frequently; avoid full-text indexing of raw message blobs.
- Implement tiered storage (hot/warm/cold) with automated rollups and downsampling.
- Separate streaming real-time detection (alerts) from ad-hoc historical search (analytics) to limit CPU/IO costs.

## Minimal, pragmatic stack (cost-effective)
- Collector: Vector (or Fluent Bit) on hosts / log-shippers at edge — structured JSON preferred. These are lightweight, fast, and configurable for filtering and sampling.
- Message bus / buffer: Redpanda (Kafka API compatible) or Kafka if you need heavy ecosystem; use for backpressure, replay and stream processing.
- Stream processor / rule engine: Lightweight Go or Rust service(s) consuming from Redpanda to evaluate streaming, low-latency rules (authentication failures, threshold-based counts). Keep rules declarative (YAML) and evaluated in-stream.
- Hot analytics store: ClickHouse (columnar) — excellent compression, high query performance, and cost-effective on commodity VMs. Use materialized views for common rollups.
- Cold/warm object store: S3-compatible (MinIO on-prem or cloud S3) with Parquet/ORC files partitioned by date and key fields.
- Query/UI: Grafana (ClickHouse datasource) or a tiny custom UI. Avoid heavy, full-log viewers that try to show TBs at once.
- Alerting: Prometheus Alertmanager-compatible webhooks + dedup/grouping layer; send to webhooks, email, or ticketing.
- Observability & Ops: Prometheus + Grafana for system metrics; use Loki for aggregator logs only if needed.

Most of the above components are Apache/BSD/MIT-licensed or offer permissive options.

## Key design decisions and why
1. Structured logs at source
   - Enforce JSON/structured output where possible. Reduces parsing costs, makes efficient columnar storage possible, and avoids expensive unstructured indexing.
   - Filter non-security fields at the collector (debug logs, verbose traces). Use sampling for chatty sources.

2. Tiered storage
   - Hot (ClickHouse): last 7–14 days of full-fidelity, indexed fields for fast searches and real-time investigations.
   - Warm (Parquet on S3): 30–90 days, columnar compressed files, queryable by batch jobs or DuckDB-like engines for less frequent queries.
   - Cold: archive snapshots (Parquet/GZIP) for long-term retention and compliance.

3. Indexing strategy
   - Index frequently queried fields (timestamp, src.ip, dest.ip, user.name, host.name, event.category, event.action).
   - Use ClickHouse primary ordering + sparse/min-max indices and Bloom filters for high-cardinality fields.
   - Avoid indexing entire message text — instead store message blobs compressed and search them on-demand (expensive; run with limits).

4. Rule engine split (streaming + batch)
   - Streaming rule engine: detect immediate threats (brute-force, port scans, credential stuffing) in real-time using windowed counters and stateful grouping in a streaming app.
   - Batch/correlation rules: run periodic queries on ClickHouse materialized views for complex correlation (multi-step attacks) and lower frequency alarms.
   - Keep rule definitions declarative and version-controlled.

5. Data reduction techniques
   - Sampling: apply controlled sampling rates to noisy telemetry (e.g., application debug logs) at collectors.
   - Deduplication: detect and drop duplicate events upstream using deterministic hashing.
   - Flatten and normalize schema; store only required fields in hot store; keep raw payloads in object storage for on-demand rehydration.

6. Pre-compute rollups & summaries
   - Maintain timed rollups per entity (src.ip, user.name, host.name) to answer most analyst queries without scanning raw logs.
   - Use materialized views in ClickHouse to make common queries cheap.

7. Cost controls
   - Enforce quotas per source/tenant and rate limiting at collectors.
   - Autoscale ingestion pipeline with queue-based throttling (Redpanda) instead of unbounded ingestion spikes.
   - Use instance families with good local NVMe and high IOPS for ClickHouse, but control retention to limit storage footprint.

## Concrete configuration guidance
- Compression: ZSTD (level tuned) for Parquet; ClickHouse default codecs (ZSTD) for column compression.
- Partitioning: daily partitions for object store; monthly partitions for very cold archives.
- ClickHouse table engine: MergeTree with useful ORDER BY (e.g., (event_date, src.ip, user.name)) and appropriate TTL rules to move data to cheaper storage tiers.
- Materialized views: maintain counts, unique counts (approximate with HyperLogLog), and sessionization windows (5m/15m/1h) for common investigations.

## Retention examples (tunable)
- Hot store: 14 days full-fidelity (fast investigation window). Keep indexed fields + message blob.
- Warm store: additional 76 days in Parquet (total 90 days). Store columnar, compressed; limit indexed fields.
- Cold archive: beyond 90 days to 7 years depending on compliance (store compressed monthly Parquet snapshots).

Trade-off: shorter hot retention saves money and I/O; warm/cold reduces cost but increases query latency for older data.

## Alerting & noise reduction
- Implement an alert dedup + grouping service to collapse noisy signals (group by rule id + group_by fields + time window).
- Rate-limit alerts per rule and per observed entity to prevent alert storms.
- Provide an "evidence snapshot" (small sample of raw events) with each alert rather than the entire log blob.

## UI & analyst ergonomics (keep simple)
- A fast timeline + field-filtering UI that queries ClickHouse via Grafana or a small web app.
- Quick drill-down pulls only the relevant raw events from object storage (rehydrate on-demand) rather than pre-loading everything.
- Provide saved queries and common playbooks (investigation workflows) to reduce ad-hoc heavy queries.

## Security and governance (must-haves)
- TLS for all transport; sign/verify configuration changes; audit trail for rules edits.
- RBAC for data and UI; least privilege for access to raw data.
- Keep a `NOTICE` file and permit-list of dependencies (MIT/BSD/Apache preferred).

## Example tech stack (minimal & permissive licenses)
- Vector (ingest/collector) — fast, small footprint
- Redpanda (Kafka-compatible) — low ops overhead
- ClickHouse — hot analytics, SQL interface
- MinIO (S3) or cloud S3 — warm/cold storage
- Grafana — UI, lightweight dashboards
- Prometheus + Alertmanager — monitoring and alert pipeline
- Small Go/Rust services for rule-evaluation + alert grouping

## Avoid these common cost traps
- Storing and indexing full raw logs forever. Index only what you need and store raw in cheap cold storage.
- Allowing unlimited ingestion from noisy sources without quotas and sampling.
- Full-text search over all messages — expensive and rarely needed for initial triage.
- Using heavyweight streaming frameworks if requirements are modest — prefer small, efficient consumers.

## Roadmap notes (2025 → 2040)
- 2025 (MVP): Structured ingestion, ClickHouse hot 7–14d, Parquet warm 30–90d, streaming rule engine for immediate alerts.
- 2026–2030: Add automated enrichment (GeoIP, TI) with cache and join tables (avoid embedding TI blobs in every event). Introduce approximate algorithms (HLL, Bloom) for uniqueness/fast checks. Prototype ML anomaly detection as post-hoc scorer with human-in-the-loop.
- 2031–2040: Explore federated query and privacy-preserving analytics (secure multi-party or differential privacy) if multi-tenant environment is required. Continue to push more computation to pre-aggregation and edge to keep per-EPS costs low.

## Final verdict: "Is this a good idea?"
Yes — focusing on cost-effectiveness is a good design constraint. It forces discipline (structured logs, indexed fields only, tiered storage, pre-aggregation) that improves performance and reduces long-term ops costs. The trade-off is more upfront engineering discipline and stricter source-side controls. Expect more investment in observability, quota enforcement, and good defaults at ingestion time.

## Next steps (pick one)
- A: I draft a short sizing matrix and sample node counts for target EPS levels (10k, 100k, 1M).  
- B: I produce a one-page rule-engine design (YAML rule DSL + streaming semantics).  
- C: You confirm EPS, retention targets, and target users (single org vs multi-tenant) and I firm up cost estimates.

Pick A, B, or C (short). If you want A or C, give target EPS and retention window preferences.