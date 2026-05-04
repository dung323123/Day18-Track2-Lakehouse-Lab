# Bonus Challenge — Lakehouse Architecture Brief

## 1) Problem statement
We need a production data platform for LLM observability at **1B requests/day** (~5 KB/request, about **5 TB/day raw JSON**).
Constraints:
- Dashboard by tenant/model/endpoint, refresh every **5 minutes**.
- Full prompt/response retained **7 days** for incident forensics.
- After 7 days, only curated aggregates retained **1 year**.
- PII must be redacted **before analyst access**.
- Storage budget cap: **<= $5,000/month**.

Why hard: this is not only scale; it is a conflicting objective set (freshness, governance, and hard FinOps cap). The wrong physical layout can satisfy one SLO and fail the other three.

## 2) Architecture diagram
```text
               +--------------------+
Ingress (HTTP) | API Gateway / LB   |
               +---------+----------+
                         |
                         v
               +--------------------+
               | Kafka / Kinesis    |  (partition by tenant_id)
               +---------+----------+
                         |
                         v
+------------------------------------------------------------------+
| BRONZE (Delta, object storage S3-compatible)                     |
| path: bronze/llm_calls_raw/date=YYYY-MM-DD/hour=HH/             |
| columns: request_id, ts, tenant_id, raw_json, ingest_ts         |
| - append only, schema evolution controlled                       |
| - PII tokenization UDF at ingest (phone/email/user identifiers)  |
+----------------------------+-------------------------------------+
                             |
                             | Structured Streaming + DQ rules
                             v
+------------------------------------------------------------------+
| SILVER (Delta)                                                     |
| path: silver/llm_calls/date=YYYY-MM-DD/                           |
| typed cols: model, endpoint, status, latency_ms, token_in/out     |
| - dedup by request_id (latest ingest_ts)                          |
| - malformed payload quarantine                                     |
| - CDC enabled for incremental downstream                           |
+----------------------------+-------------------------------------+
                             |
                             | 5-min micro-batch aggregation
                             v
+------------------------------------------------------------------+
| GOLD (Delta)                                                       |
| path: gold/llm_5min_metrics/date=...                              |
| keys: window_start, tenant_id, model, endpoint                    |
| metrics: p50/p95 latency, error_rate, token_in/out, cost_usd      |
| - optimized + clustered by (tenant_id, model)                     |
+----------------------------+-------------------------------------+
                             |
                             +--> BI dashboard (Trino/Databricks SQL)
                             +--> Alerting (SLO breaches)

Governance plane:
- Catalog: Apache Polaris (open REST catalog)
- Lineage: OpenLineage + Marquez
- Access: RBAC + column masking policy on sensitive fields
```

## 3) Key decisions with rejected alternatives
### Decision A: Table format = Delta Lake
- Chosen: **Delta** for mature MERGE, time travel, and operational ergonomics with compaction + data skipping.
- Rejected Iceberg: strong ecosystem, but our team already has Delta operations know-how; migration risk and retraining cost are non-trivial in 6-week horizon.
- Rejected Hudi: excellent upsert profile, but governance/query interoperability in our target toolchain is weaker for this team.

### Decision B: Medallion + strict contract per layer
- Chosen: Bronze raw immutable, Silver typed/deduped, Gold business metrics.
- Rejected “single big curated table”: faster to bootstrap but creates inconsistent query logic, poor auditability, and hard rollback.
- Rejected “raw only + semantic layer in BI”: shifts data quality burden to every consumer, multiplies logic drift.

### Decision C: Partitioning strategy
- Chosen: Bronze partition by `date/hour`; Silver and Gold partition by `date`, cluster/Z-order by `tenant_id, model`.
- Rejected partition by `tenant_id`: high-cardinality directories, metadata explosion.
- Rejected no partition: excessive scan for time-bounded dashboards and retention jobs.

### Decision D: Retention and tiering
- Chosen: Bronze/Silver 7-day hot retention (S3 Standard), Gold 1-year warm retention (S3 IA for partitions older than 30 days).
- Rejected keeping raw 1 year: violates cost cap immediately.
- Rejected Glacier for Gold early: retrieval latency conflicts with ad-hoc analytics and incident reviews.

### Decision E: Catalog and governance
- Chosen: **Apache Polaris** as vendor-neutral catalog, with policy enforcement and lineage integration.
- Rejected Unity-only catalog lock-in: operationally convenient short-term, but conflicts with long-term portability mandate.
- Rejected “no central catalog”: causes schema drift, weak discoverability, and no authoritative ownership model.

### Decision F: Ingestion mode
- Chosen: streaming ingest + 5-minute micro-batch Gold updates.
- Rejected hourly batch: cheaper compute but fails 5-minute dashboard freshness.
- Rejected fully real-time Gold row-level writes: unnecessary compute overhead for percentile metrics.

## 4) Failure modes, detection, rollback
### FM1: Bad schema rollout from producer (e.g., latency_ms becomes string)
- Detect: Silver expectation checks fail (`latency_ms` parse null-rate > threshold).
- Immediate action: route bad events to quarantine table, keep pipeline alive.
- Rollback: use Delta time travel to restore last good Silver snapshot, redeploy parser with dual-read fallback.

### FM2: Duplicate storm from client retries (idempotency bug)
- Detect: `count(*) / count(distinct request_id)` spike in Bronze and Silver.
- Immediate action: tighten dedup watermark window in Silver.
- Rollback: recompute affected Silver partitions from Bronze for impacted hours, then re-materialize Gold windows.

### FM3: Cost explosion due to compaction misconfiguration
- Detect: daily compute burn > budget envelope; optimize jobs exceed expected runtime.
- Immediate action: disable aggressive compaction schedule; cap OPTIMIZE to high-value partitions.
- Rollback: revert scheduler config and replay only required partitions.

### FM4: Accidental bad data publish to Gold
- Detect: anomaly rule (p95 latency drops unrealistically across all tenants).
- Immediate action: freeze BI refresh pointer.
- Rollback: `RESTORE` Gold to previous version and replay last 2 windows from Silver CDC.

## 5) Back-of-envelope cost estimate
Assumptions:
- Raw ingress: 5 TB/day.
- Compression + columnar conversion reduces Silver footprint by ~35% from raw-equivalent.
- Gold is small (aggregates), assume 1.5% of Silver volume.

### Storage
- Bronze 7 days: `5 TB/day * 7 = 35 TB` on Standard.
- Silver 7 days: `3.25 TB/day * 7 = 22.75 TB` on Standard.
- Gold 365 days: `0.04875 TB/day * 365 ≈ 17.8 TB` mostly IA.

Pricing model (rounded, region-agnostic planning numbers):
- Standard: **$23/TB-month**
- IA: **$12.5/TB-month**

Monthly equivalent:
- Bronze Standard: `35 * 23 = $805`
- Silver Standard: `22.75 * 23 = $523`
- Gold IA: `17.8 * 12.5 = $223`
- Metadata/log overhead + replicas (~20%): `~$310`

Estimated storage subtotal: **~$1,861/month**.

### Compute (ingest + silver + gold + optimize)
- Streaming + ETL cluster envelope: **~$2,400/month**
- Daily optimize + maintenance jobs: **~$450/month**
- BI query/serving budget: **~$600/month**

Estimated compute subtotal: **~$3,450/month**.

### Total
- Storage + compute: **~$5,311/month** (over budget).

FinOps adjustment to hit cap:
1. Reduce Bronze full payload from 7 days to 5 days for low-priority tenants (policy exception) with encrypted offload of minimal forensic fields.
2. Lower optimize cadence on cold partitions (every 24h -> every 72h).
3. Enforce tenant-level retention class and sampling for non-critical debug fields.

Expected post-adjustment: **~$4,850–$4,980/month**.

## 6) One-week MVP slice
Goal: prove architecture viability with minimum shippable slice.

Day 1-2:
- Build Bronze ingest from stream with tokenization UDF.
- Define Delta tables and catalog registration.

Day 3:
- Implement Silver parse + dedup + quarantine.
- Add 3 quality assertions (schema, null-rate, duplicate ratio).

Day 4:
- Implement Gold 5-minute aggregates (p50/p95/error_rate/cost).
- Add dashboard prototype for top 20 tenants.

Day 5:
- Add rollback runbook: restore Gold, replay from Silver CDC.
- Run game-day test with injected schema failure + duplicate storm.

Definition of done:
- 5-minute dashboard refresh achieved for pilot tenants.
- At least one validated rollback in < 15 minutes.
- Daily cost report generated with projected month-end budget.

## 7) Why this design is defensible
This design is defensible because each hard constraint is mapped to a concrete mechanism: freshness via micro-batch Gold, reliability via Delta time-travel/restore, governance via tokenization + catalog policy, and cost via explicit tiering and optimize boundaries. It is not “best practice by slogan”; it is a constrained choice with explicit trade-offs and rejected alternatives.
