# Statement of Work — "Project HARVEST"
## End-to-End Data Platform on Google Cloud
### Multi-Source Batch + Streaming Ingestion, Lakehouse Modelling, and Automated Delivery

| Field | Value |
|---|---|
| Document type | Statement of Work / Solution Design Document (SDD) |
| Version | 1.0 (Design Baseline) |
| Client (simulated) | FreshCart Retail Pvt. Ltd. — quick-commerce grocery |
| Delivery model | Single engineer, full lifecycle (architect + build + deploy) |
| Target environment | GCP, single billing account, free-trial credit constrained |
| Duration | 8 phases, ~6–8 weeks part-time |

---

## 0. How to use this document

This is written the way a real SOW/SDD lands on your desk in an enterprise engagement: business context first, then scope, then design, then delivery plan. Every major technology choice has a **Trade-off** block explaining what you gained, what you gave up, and what you would choose differently at 100x the data volume. That reasoning is the actual deliverable — anyone can wire services together, but being able to defend *why* is what makes you the architect in the room.

Read it once end to end before touching a console. Then we work phase by phase.

---

## 1. Business context (the scenario)

FreshCart is a quick-commerce grocery player operating dark stores across 6 Indian cities. Today the business runs on disconnected systems and every question takes 3 days and a data analyst with a CSV export. Leadership wants a single analytical platform.

**The five business questions the platform must answer:**

| # | Question | Latency requirement | Drives |
|---|---|---|---|
| Q1 | What is our real-time order volume, GMV, and cancellation rate by city and dark store? | < 5 minutes | Live ops dashboard |
| Q2 | Which SKUs are trending toward stockout in the next 4 hours? | < 15 minutes | Inventory replenishment |
| Q3 | What is our customer cohort retention and repeat-purchase behaviour? | Next-day | Growth / CRM |
| Q4 | What is true unit economics per order, including partner delivery cost and FX-adjusted imported goods cost? | Next-day | Finance |
| Q5 | Where do users drop out of the funnel (app browse → cart → checkout → paid)? | Hourly | Product |

**Why this scenario is good practice:** it forces genuinely different ingestion patterns — a change-data-capture stream, a high-volume event stream, partner flat files, an external API, and human-maintained reference data. A pipeline with one source teaches you one thing. This teaches you five, plus the hard part: reconciling them into one conformed model.

---

## 2. Objectives and success criteria

**Objectives**

1. Ingest five heterogeneous sources into a governed GCP lakehouse.
2. Produce a conformed, tested, documented dimensional model serving the five business questions.
3. Deliver everything as code — infrastructure, transformations, orchestration, tests — deployed via automated CI/CD from GitHub with no manual console changes in the promoted environments.
4. Operate within free-trial credits with hard cost guardrails.

**Success criteria (acceptance)**

| ID | Criterion | How verified |
|---|---|---|
| AC-1 | All five sources land in the raw zone on schedule with zero manual intervention for 7 consecutive days | Orchestrator run history |
| AC-2 | Streaming path end-to-end latency (event emitted → queryable in serving layer) under 5 minutes at p95 | Latency instrumentation table |
| AC-3 | Every mart table has data-quality tests that fail the pipeline on breach | CI run + orchestrator failure logs |
| AC-4 | A commit to `main` deploys infrastructure and transformation changes to prod with no human console access | GitHub Actions run |
| AC-5 | Full environment can be destroyed and rebuilt from code in under 45 minutes | `terraform destroy` → `apply` timed |
| AC-6 | Total spend stays under the credit budget with automated alerting at 50/75/90% | Billing budget alerts |
| AC-7 | Every dimension and fact table is documented with lineage discoverable in the catalog | Dataplex / dbt docs |

AC-5 is the one people skip and the one that separates a demo from a platform. Insist on it.

---

## 3. Scope

**In scope**

- Ingestion, storage, transformation, orchestration, quality, catalog, serving layer, IaC, CI/CD, monitoring, cost governance, runbook documentation.
- Three logical environments: `dev`, `stg`, `prod` (see §12 for how to do this without triple cost).
- Synthetic data generation tooling (this substitutes for the real source systems).

**Out of scope**

- BI dashboard design beyond one reference Looker Studio dashboard.
- ML models / feature store. (Deliberately excluded — noted in §18 as the natural v2.)
- Multi-region DR, VPC Service Controls perimeter, CMEK. Designed for, not implemented — the design discussion is in §14.
- Real PII. All data is synthetic; PII handling patterns are implemented against synthetic values.

**Assumptions**

- Single GCP billing account with trial credits.
- You have owner-level access on the project.
- GitHub account available; GitHub Actions free tier sufficient.

---

## 4. Source system inventory

This is the table an architect writes first. Everything downstream is derived from it.

### S1 — Operational OLTP database (CDC)
| Attribute | Value |
|---|---|
| System | Cloud SQL for PostgreSQL — `freshcart_ops` |
| Entities | `orders`, `order_items`, `customers`, `stores`, `riders` |
| Pattern | Change Data Capture, log-based (logical replication) |
| Volume (simulated) | ~50k rows seeded, ~2k mutations/day |
| Latency need | Near real-time (Q1, Q2) |
| Nasty realities to model | Late-arriving updates, hard deletes, schema evolution, `updated_at` you cannot trust |

### S2 — Application clickstream (streaming events)
| Attribute | Value |
|---|---|
| System | Mobile/web app → Pub/Sub topic `clickstream-events` |
| Format | JSON events: `app_open`, `search`, `product_view`, `add_to_cart`, `checkout_start`, `payment_success`, `payment_fail` |
| Volume (simulated) | 5–20 events/sec, generated in bursts, not continuously |
| Latency need | < 5 min (Q1, Q5) |
| Nasty realities | Out-of-order events, duplicates (at-least-once delivery), mobile clients buffering offline and replaying hours later, bot traffic |

### S3 — Partner / 3PL flat files (batch)
| Attribute | Value |
|---|---|
| System | Delivery partner drops files to a GCS landing bucket (simulates SFTP) |
| Format | Daily CSV `delivery_costs_YYYYMMDD.csv`, weekly semi-structured JSON `rider_shifts_*.json` |
| Volume | ~10–50 MB/day |
| Latency need | Next-day (Q4) |
| Nasty realities | Files arrive late or twice, column order changes without notice, encoding issues, header-only empty files, ₹ symbols and thousands separators in numeric columns |

### S4 — External REST API (scheduled pull)
| Attribute | Value |
|---|---|
| System | Public FX rates API + a public weather API |
| Format | JSON over HTTPS, rate-limited, API key |
| Volume | Tiny — a few hundred records/day |
| Latency need | Daily (Q4), hourly for weather (demand correlation) |
| Nasty realities | Rate limits, 5xx flakiness, silent schema drift, key rotation, backfilling gaps after an outage |

### S5 — Reference / master data (human-maintained)
| Attribute | Value |
|---|---|
| System | Google Sheet maintained by the category team — product hierarchy, city→region mapping, promo calendar |
| Format | Sheet → BigQuery external table |
| Volume | Hundreds of rows |
| Latency need | Daily |
| Nasty realities | Humans. Blank rows, merged cells, renamed tabs, a "notes" column someone types dates into. Needs SCD Type 2 because the hierarchy changes and history matters |

**Why five sources, deliberately:** each one maps to a different GCP ingestion service and a different failure mode. When your new employer hands you a source, you will have already seen its shape.

---

## 5. Target architecture

```
                          ┌──────────────────────────────────────────────┐
                          │              GOVERNANCE & OPS                │
                          │  Dataplex catalog · Cloud Monitoring         │
                          │  Cloud Logging · Billing budgets · IAM       │
                          └──────────────────────────────────────────────┘

 SOURCES              INGESTION              STORAGE / PROCESSING           SERVING

 S1 Cloud SQL ──► Datastream (CDC) ─────►┐
                                          │
 S2 App events ──► Pub/Sub ──► Dataflow ─┤
                    │          (streaming)│
                    │                     ├──► GCS Raw Zone ──► BigQuery
 S3 Partner files ──► GCS landing         │    (Parquet/Avro)   ┌─────────┐
                    │  └► Cloud Function  │                     │ BRONZE  │  raw, append-only
                    │     (trigger)       │                     ├─────────┤
                    │                     │                     │ SILVER  │  cleaned, conformed,
 S4 External API ──► Cloud Run Job ───────┤                     │         │  deduped, SCD2
                    │  (Scheduler-driven) │                     ├─────────┤
                    │                     │                     │  GOLD   │  star schema marts
 S5 Google Sheet ──► BQ external table ───┘                     └─────────┘
                                                                     │
                          Dataproc Serverless (Spark)                 ├─► Looker Studio
                          — heavy file parsing, reprocessing          ├─► BI Engine
                                                                     └─► Reverse ETL
                          dbt Core on Cloud Run                          (Cloud Run → API)
                          — SILVER→GOLD SQL, tests, docs

                    ╔══════════════════════════════════════════╗
                    ║  ORCHESTRATION: Cloud Composer (Airflow)  ║
                    ║  + Cloud Scheduler + Workflows            ║
                    ╚══════════════════════════════════════════╝

                    ╔══════════════════════════════════════════╗
                    ║  DELIVERY: GitHub → Actions → Terraform   ║
                    ║  Workload Identity Federation (keyless)   ║
                    ╚══════════════════════════════════════════╝
```

### 5.1 Zone contract

| Zone | Storage | Format | Mutability | Retention | Purpose |
|---|---|---|---|---|---|
| Landing | GCS `gs://fc-landing-{env}` | Native (CSV/JSON as received) | Immutable | 30 days | Byte-exact copy of what arrived. Never transform here. |
| Raw / Bronze | GCS + BigQuery `bronze_*` | Parquet on GCS, native BQ for streams | Append-only | 90 days | Typed, partitioned, source-shaped. One table per source entity. Includes ingestion metadata columns. |
| Silver | BigQuery `silver_*` | Native BQ | Merge/upsert | 400 days | Deduplicated, conformed, business keys resolved, SCD2 applied, quality-tested. |
| Gold | BigQuery `gold_*` | Native BQ | Rebuilt/incremental | Indefinite | Star schema. Facts + conformed dimensions. What BI touches. |

**Non-negotiable rule:** BI tools and stakeholders touch Gold only. If someone queries Bronze, your model has failed and you will spend the rest of your tenure answering "why is this number different."

### 5.2 Mandatory metadata columns on every Bronze table

```
_ingested_at        TIMESTAMP   -- when we wrote it
_source_system      STRING      -- S1..S5
_source_file        STRING      -- GCS URI or Pub/Sub message id
_batch_id           STRING      -- orchestrator run id, for surgical reprocessing
_payload_hash       STRING      -- for idempotent dedup
_schema_version     STRING      -- detected contract version
```

Add these on day one. Retrofitting lineage columns after you have 200 GB is a genuinely miserable week.

---

## 6. Service selection and trade-offs

This is the heart of the document. In an interview or a design review, this section is what you are actually being tested on.

### 6.1 S1 ingestion — Datastream vs alternatives

**Chosen: Datastream → GCS (Avro) → BigQuery, with a BQ `MERGE` in Silver.**

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| **Datastream** | Managed log-based CDC, no agent, handles DDL drift, serverless, direct BQ destination available | Costs per GB processed, limited source support, less control over transform-in-flight | **Chosen.** It is the GCP-native answer and what an enterprise would use. |
| Debezium on Dataproc/GKE + Kafka | Full control, portable, huge ecosystem | You now operate Kafka and Debezium. Cost and ops burden are enormous for one engineer | Rejected — but know this pattern, it is common in mature shops |
| Scheduled batch extract (`SELECT * WHERE updated_at > x`) | Trivial to build, near-zero cost | Misses hard deletes, misses intra-window updates, hammers the OLTP DB, `updated_at` is often wrong | Rejected — but this is what 70% of real pipelines actually do, and knowing *why* it is wrong is valuable |
| Federated queries (BigQuery → Cloud SQL) | No pipeline at all | Load on OLTP, no history, no scale | Rejected for facts; acceptable for tiny lookups |

**Trade-off you must be able to articulate:** Datastream gives you the log, not the truth. You still have to build the merge logic that turns an insert/update/delete stream into a current-state table, and decide whether Silver holds current state (Type 1) or full history (Type 2). We do **both**: `silver_orders_current` and `silver_orders_history`.

**Free-trial note:** Datastream bills per GB. With a seeded 50k-row database and 2k mutations/day this is pennies. Do not seed 50 million rows because it "feels more production." Volume teaches you nothing that 50k rows plus correct partitioning does not.

### 6.2 S2 ingestion — streaming compute

**Chosen: Pub/Sub → Dataflow (Apache Beam, Python) → BigQuery Storage Write API.**

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| **Dataflow** | True streaming semantics — event-time windows, watermarks, late-data handling, exactly-once to BQ. Autoscaling. The correct tool. | Always-on workers cost real money. Slowest to develop in. Beam has a learning curve. | **Chosen for the design**, run in bounded windows to control cost (see below) |
| Pub/Sub BigQuery Subscription | Zero code, zero cost beyond Pub/Sub, direct to BQ | No transformation, no enrichment, no windowing, no dedup, schema must match exactly | **Also implemented** — as the cheap "raw firehose" path in parallel |
| Dataflow templates (Google-provided) | No Beam code needed | Inflexible | Used for a first smoke test only |
| Cloud Run + push subscription | Cheap, familiar, scales to zero | You hand-roll ordering, windowing, retries, dedup. Fine for simple enrichment, wrong for aggregation | Rejected for main path |

**The cost-conscious design decision:** we implement *both* paths and make it an explicit architectural point.
- **Path A (always on, near-free):** Pub/Sub BigQuery subscription → `bronze_events_raw`. Runs 24/7.
- **Path B (Dataflow, run on demand):** the sessionization + enrichment + dedup job. You run it for a few hours during Phase 4, prove p95 latency, capture screenshots and metrics, then tear it down and replace with a scheduled micro-batch in BigQuery SQL over `bronze_events_raw`.

Do not leave a Dataflow streaming job running unattended on trial credits. A single `n1-standard-2` worker running all month is roughly the cost of a small server; three autoscaled workers will burn a meaningful slice of a $300 credit before you notice. **Set a hard `--maxNumWorkers` and a budget alert before you launch it.** Verify current pricing on the GCP pricing calculator before you start — rates change.

**Streaming concepts to prove you understand, implemented in the Beam job:**
- Event time vs processing time, and why a user's phone in a tunnel breaks your daily aggregate.
- Fixed vs sliding vs **session** windows (sessionization of clickstream is the classic use case).
- Watermarks and allowed lateness; where late events go (a side output to a `late_events` table — never silently dropped).
- Deduplication on `event_id` within a window.

### 6.3 S3 ingestion — file processing

**Chosen: GCS event trigger → Cloud Function (validation + routing) → Dataproc Serverless Spark (parsing) for the messy weekly JSON; direct BQ load for the well-formed daily CSV.**

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| Direct `bq load` from GCS | Free, fast, simple | No transformation, fails hard on dirty data | **Chosen for clean CSV** |
| **Dataproc Serverless** | Real Spark, your existing skill, no cluster to manage, per-second billing, scales to zero | Cold start ~60s, min billing quantum, more expensive than BQ SQL for the same work | **Chosen for messy JSON** — and specifically to exercise your Spark skill on GCP |
| Dataproc cluster (persistent) | Fast start, interactive, Jupyter | You pay for idle VMs. Death to trial credits. | Rejected — but create one for 30 minutes in Phase 3 just to see it, then delete it |
| Dataflow batch | Consistent with streaming path | Beam batch is more work than Spark for the same job when you already know Spark | Rejected |
| BigQuery external table + SQL | Cheapest possible | Cannot handle deeply nested/malformed JSON gracefully | Rejected for this source |

**Trade-off to articulate:** "We used Spark not because the volume required it, but because the *transformation complexity* required it. Volume drives cluster sizing; complexity drives engine choice." This is a genuinely good line in a design review, and it is true.

**Cloud Function responsibilities (the ingestion gatekeeper pattern):**
1. Validate filename against expected pattern and date.
2. Check file is non-empty and header matches the registered schema contract.
3. Compute checksum, check against a `file_registry` BQ table — reject duplicate re-drops (idempotency).
4. Move to `landing/valid/` or `landing/quarantine/` with a reason.
5. Write a row to `ingestion_audit`.
6. Publish a completion message to Pub/Sub that the orchestrator sensor listens for.

Quarantine-don't-drop is a production behaviour. Build it now.

### 6.4 S4 ingestion — API pull

**Chosen: Cloud Run Job, triggered by Cloud Scheduler.**

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| **Cloud Run Job** | Runs to completion, no HTTP server needed, up to 24h, containerized (portable, testable locally), scales to zero | Container build step in CI | **Chosen** |
| Cloud Function (2nd gen) | Simplest deploy | 60-min ceiling, awkward for long paginated pulls, less portable | Reasonable alternative; use for the hourly weather pull to practice both |
| Airflow `PythonOperator` in Composer | One less service | Puts business logic in the orchestrator. Orchestrators should orchestrate, not compute. | Rejected on principle — and say exactly that in a review |

**Production behaviours to implement here (this is where junior pipelines break):**
- Exponential backoff with jitter on 429/5xx.
- Idempotent writes keyed on `(api_name, business_date)` so a rerun overwrites rather than duplicates.
- **Backfill mode:** the job accepts a date range parameter so you can replay a week of missed pulls. Design every batch job to be re-runnable for an arbitrary date. If a job can only run for "today," it is not production code.
- Secret in Secret Manager, never in env vars or the repo.
- Schema drift detection: hash the response's key structure, alert if it changes.

### 6.5 S5 ingestion — reference data

**Chosen: BigQuery external table over Google Sheets → daily snapshot into Bronze → SCD Type 2 in Silver.**

Trade-off: external tables are queried live, so a mid-query edit by a human gives you an inconsistent read. Hence the daily snapshot — pin the state, then process. This is a small detail that demonstrates real thinking.

**SCD Type 2 is mandatory here.** When "Beverages" gets split into "Beverages — Hot" and "Beverages — Cold" in March, Q3's cohort report must still be reproducible. Implement with `valid_from`, `valid_to`, `is_current`, and a surrogate key, via BigQuery `MERGE`.

### 6.6 Transformation layer

**Chosen: dbt Core, executed in a container on Cloud Run Jobs, orchestrated by Airflow.**

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| **dbt Core** | Version-controlled SQL, DAG-aware, built-in testing, auto docs + lineage, incremental models, environment targets. Industry standard. | Another tool to learn (worth it — it's ~2 days) | **Chosen** |
| Raw SQL in Airflow `BigQueryInsertJobOperator` | No new tool | No lineage, no tests, no docs, dependency order maintained by hand | Rejected — but implement 2–3 models this way in Phase 5 so you have felt the pain |
| Dataform | GCP-native, no container needed, free | Smaller ecosystem, less portable, less common on job specs | Strong alternative. **Build one model in Dataform** so you can compare them in an interview. |
| Spark for all transforms | One engine everywhere | Vastly more expensive and slower than BigQuery for set-based SQL | Rejected |

**The ELT vs ETL point:** we deliberately do **ELT** — land raw, transform in-warehouse — because BigQuery's compute is separated from storage and is cheaper and faster at set-based work than any cluster you would run. We use Spark only where the source is not tabular. Being able to say *when ETL still wins* (heavy non-SQL logic, data that must be cleansed before it can legally land, egress-constrained sources) is the mark of someone who understands the trade rather than repeating a slogan.

### 6.7 Orchestration

**Chosen: Cloud Composer 3 (managed Airflow) as the design target, with a staged cost-managed rollout.**

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| **Cloud Composer** | Managed Airflow — the single most in-demand orchestration skill on data engineering job specs. You already know Airflow. | **Expensive.** Even a small environment runs a persistent GKE cluster and costs on the order of hundreds of dollars a month. | **Chosen for the design and for a time-boxed 5–7 day proof.** |
| Self-hosted Airflow on one small GCE VM | ~$15–25/month, full Airflow | You manage upgrades, the DB, and reliability | **Chosen for the long development period.** Same DAG code. |
| Cloud Workflows + Cloud Scheduler | Serverless, near-free, GCP-native | YAML, weak for complex dependency graphs, no backfill/catchup, no rich UI | **Chosen for the "ingestion trigger" sub-flows** — a genuinely appropriate use, not just a cost dodge |
| Airflow in Docker locally | Free, fast iteration | Not a deployment | **Chosen for development** |

**The staged approach — and this is a real architectural pattern, not a shortcut:**
1. Develop all DAGs locally in Docker Compose (free, fast).
2. Run the platform day to day on self-hosted Airflow on an `e2-small` VM, deployed by Terraform.
3. In Phase 7, stand up Cloud Composer, deploy the identical DAG bundle via CI/CD, run for 5–7 days, capture the evidence, then destroy it.

You end up having genuinely deployed to Composer, having done it via automation, and having spent a small fraction of the credit. In a design review you defend it as "we validated portability of the DAG bundle across a self-managed and a managed Airflow runtime" — which is exactly what you did.

**DAG design principles to enforce:**
- One DAG per source ingestion + one DAG per transformation domain. Not one monolith.
- Every task idempotent and parameterized on `{{ ds }}`. Reruns must be safe.
- Cross-DAG dependency via Datasets (Airflow 2.4+) or sensors, not by guessing at schedules.
- Task groups for readability; dynamic task mapping for the per-file fan-out.
- SLA misses and failures route to a real alert channel, not just the UI.
- **No business logic in the DAG.** Operators call Cloud Run Jobs, dbt, or BigQuery. The DAG describes order and dependency, nothing else.

### 6.8 Serving

- **BigQuery Gold layer** with partitioning (`DATE` on the fact's event date) and clustering (`store_id, city`).
- **Materialized views** for the live-ops aggregates behind Q1 — auto-refreshing, and BigQuery rewrites queries to use them transparently. Discuss the trade-off vs scheduled queries: MVs are lower-latency and self-maintaining but restricted in what SQL they permit.
- **BI Engine reservation** (a small one) to demonstrate sub-second dashboard response, then release it.
- **Looker Studio** dashboard as the reference consumer (free).
- **Reverse ETL stub:** a Cloud Run service exposing the stockout-risk table (Q2) as a JSON API, showing that a warehouse can serve operational systems, not just dashboards.


---

## 7. Dimensional model (Gold layer)

Star schema. Kimball. Not because it is fashionable but because BI tools, and humans, reason about it correctly.

**Facts**

| Table | Grain | Type | Partition | Cluster |
|---|---|---|---|---|
| `fct_order_line` | One row per order line item | Transaction | `order_date` | `store_key, product_key` |
| `fct_order_status_event` | One row per status transition (placed→packed→dispatched→delivered/cancelled) | Accumulating snapshot alternative — kept as event grain | `event_date` | `order_key` |
| `fct_clickstream_event` | One row per app event | Transaction | `event_date` | `session_key, event_type` |
| `fct_session` | One row per user session | Periodic aggregate | `session_date` | `customer_key` |
| `fct_delivery_cost` | One row per delivery per partner | Transaction | `delivery_date` | `partner_key` |
| `fct_inventory_snapshot` | One row per SKU per store per hour | Periodic snapshot | `snapshot_date` | `store_key, product_key` |

**Dimensions**

| Table | SCD type | Note |
|---|---|---|
| `dim_customer` | Type 2 | Address/city changes must not rewrite history |
| `dim_product` | Type 2 | Category hierarchy from S5 changes over time |
| `dim_store` | Type 2 | Dark stores open, close, get re-zoned |
| `dim_date` | Static | Generated, with Indian fiscal year, festival flags, weekend flags |
| `dim_time_of_day` | Static | Quick-commerce is intensely time-of-day driven |
| `dim_delivery_partner` | Type 1 | History not analytically meaningful |
| `dim_promo` | Type 2 | From S5 promo calendar |

**Design points worth defending:**
- Surrogate keys everywhere (`INT64` generated, not natural keys) so source-system key collisions and re-keying cannot break the model.
- A late-arriving-dimension strategy: an order for a product not yet in `dim_product` gets an inferred member row rather than being dropped or failing the load.
- A "unknown" member (`-1`) in every dimension. Never a `NULL` foreign key.
- Denormalize the product hierarchy into `dim_product` rather than snowflaking. BigQuery joins are cheap but analysts' patience is not.
- Consider `ARRAY<STRUCT>` for order lines nested inside orders as a BigQuery-native alternative — and be able to explain the trade-off: nested is cheaper to scan and avoids a join, but is harder for BI tools and less portable. We keep flat facts for the marts and use nested structures in Silver.

---

## 8. Data quality and governance

**Three layers of testing, each catching something different:**

1. **Contract validation at ingestion** (Cloud Function / Beam) — is this file/message even the shape we agreed? Reject to quarantine.
2. **dbt tests at transformation** — `unique`, `not_null`, `accepted_values`, `relationships` on every model; custom singular tests for business rules (order total must equal sum of lines; no negative quantities; no delivery timestamp before order timestamp).
3. **Reconciliation checks** — source row count vs Bronze vs Silver, daily, written to a `dq_results` table and dashboarded. Financial totals reconciled to source within a tolerance. This is what auditors and finance actually ask for.

**Severity model:** not every failure should stop the pipeline. Define `error` (halt the DAG) vs `warn` (log, alert, continue). A null in a nullable descriptive field is a warn. A duplicate primary key in a fact is an error. Getting this distinction wrong is why people disable alerts.

**Governance**
- **Dataplex** — register the lakehouse, attach the GCS zones and BQ datasets, run auto data-profiling and data-quality scans, and get a searchable catalog with lineage. BigQuery also emits lineage natively — enable it.
- **Tagging / labels** — every resource labelled `env`, `owner`, `cost_center`, `data_domain`. This is how cost attribution and access reviews work at scale, and it takes ten seconds in Terraform.
- **PII handling pattern** — synthetic PII columns (email, phone) get: column-level security via policy tags in Data Catalog, a masked view for general analysts, and a documented retention rule. Demonstrating the pattern matters more than the data being real.

---

## 9. Security and IAM design

- **Service account per workload**, never one shared account, never user credentials in automation. `sa-ingest-api`, `sa-dataflow`, `sa-dbt`, `sa-composer`, `sa-terraform`.
- **Least privilege, custom roles where predefined roles are too broad.** `roles/bigquery.dataEditor` on a specific dataset, not project-wide.
- **No service account keys anywhere.** GitHub authenticates via **Workload Identity Federation** — GitHub's OIDC token is exchanged for short-lived GCP credentials. If you take one security practice from this project into your job, take this one; downloaded JSON keys in CI are the single most common serious misconfiguration in real environments.
- **Secret Manager** for API keys and DB credentials, accessed at runtime by the workload's SA.
- **Network:** Cloud SQL with private IP, Dataflow workers with no external IPs, Private Google Access. Designed and implemented where free.
- **Designed but not implemented** (document the design, note the cost/complexity reason): VPC Service Controls perimeter, CMEK for BigQuery and GCS, org policy constraints, Access Transparency. Know what each does and when you would need it — regulated clients will ask.

---

## 10. Observability and operations

| Concern | Implementation |
|---|---|
| Pipeline health | Airflow SLA misses + task failure callbacks → email/Slack webhook |
| Freshness | A `dq_freshness` check per Gold table; alert if max(`_ingested_at`) exceeds SLA |
| Latency | Instrumentation table capturing event-time → queryable-time per streaming record; p50/p95/p99 dashboard |
| Cost | Billing export to BigQuery + a cost-per-pipeline dashboard using resource labels |
| Query performance | `INFORMATION_SCHEMA.JOBS` analysis — slot-hours by query, bytes scanned, worst offenders |
| Logs | Cloud Logging sinks; error-rate log-based metrics with alerting policies |
| Runbook | Markdown in the repo: how to backfill a day, how to reprocess a quarantined file, how to handle a schema change, who to call |

The runbook is a deliverable, not an afterthought. Write it as you build, not at the end.

---

## 11. Repository structure

Monorepo. One repo, clear boundaries.

```
freshcart-data-platform/
├── README.md
├── docs/
│   ├── architecture.md            # this document, maintained
│   ├── adr/                       # Architecture Decision Records
│   │   ├── 0001-cdc-with-datastream.md
│   │   ├── 0002-elt-over-etl.md
│   │   └── 0003-composer-vs-self-hosted-airflow.md
│   ├── runbook.md
│   └── data-contracts/            # JSON Schema per source
├── infra/
│   └── terraform/
│       ├── modules/               # reusable: bq_dataset, gcs_bucket, cloud_run_job, sa_iam
│       ├── envs/
│       │   ├── dev/
│       │   ├── stg/
│       │   └── prod/
│       └── backend.tf             # GCS remote state, state locking
├── ingestion/
│   ├── api_puller/                # Cloud Run Job — S4
│   ├── file_validator/            # Cloud Function — S3
│   └── event_generator/           # synthetic data producer — S2
├── streaming/
│   └── beam_clickstream/          # Dataflow pipeline
├── batch/
│   └── spark_jobs/                # Dataproc Serverless PySpark
├── transform/
│   └── dbt/
│       ├── models/{bronze,silver,gold}/
│       ├── tests/
│       ├── macros/
│       └── dbt_project.yml
├── orchestration/
│   └── dags/
├── quality/
│   └── reconciliation/
├── tests/                         # pytest — unit tests for all Python
└── .github/
    └── workflows/
        ├── ci.yml
        ├── cd-dev.yml
        ├── cd-prod.yml
        └── terraform-plan-pr.yml
```

**Architecture Decision Records are the highest-value/lowest-effort thing here.** One page each: context, options considered, decision, consequences. Write one for every trade-off in §6. When your new manager asks why you chose something, you hand them a document instead of remembering. Do this at your actual job from week one.

---

## 12. Environments and CI/CD

### 12.1 Environment strategy on a budget

Three full environments would triple your cost. Instead:

| Env | GCP realisation | Purpose |
|---|---|---|
| `dev` | Same project, `_dev` dataset suffix, `-dev` bucket suffix, smaller resources | Daily development |
| `stg` | Same project, `_stg` suffix, created and destroyed by CI on demand | Integration test on every PR merge |
| `prod` | Ideally a separate project; acceptable to use `_prod` suffix if credits are tight | The "real" pipeline |

**Document that separate projects are the correct answer** and that suffix-based separation is a deliberate, cost-driven compromise with a stated migration path. Terraform workspaces + variable files make the switch to real projects a config change, not a rewrite. Naming the compromise explicitly is what a senior engineer does; pretending it is best practice is what a junior does.

### 12.2 Pipeline design

**On pull request:**
1. Lint (`ruff`, `sqlfluff`), format check.
2. Unit tests (`pytest`) with mocked GCP clients.
3. `dbt compile` + `dbt parse` — catch broken refs without touching the warehouse.
4. `terraform fmt -check`, `terraform validate`, `terraform plan` — plan output posted as a PR comment. **Never auto-apply from a PR.**
5. Security scan: `tfsec`/`checkov` on Terraform, `trivy` on container images, secret scanning.
6. Build container images, tag with commit SHA, push to Artifact Registry.

**On merge to `main` → deploy to dev:**
7. `terraform apply` to dev.
8. Deploy Cloud Run Jobs / Functions from SHA-tagged images.
9. Sync DAGs to the Airflow DAG bucket.
10. `dbt build --target dev` against a small seeded dataset — this runs models *and* tests.
11. Integration test: trigger one full pipeline run on synthetic data, assert row counts and DQ results.

**On tag `v*` → deploy to prod:**
12. Manual approval gate (GitHub environment protection rule).
13. `terraform apply` to prod.
14. `dbt build --target prod` with `--defer` to prod state so only changed models rebuild.
15. Smoke test + automatic rollback path documented.

**Key practices to implement because they are what real teams grade you on:**
- **Immutable artifacts.** Build once, promote the same SHA-tagged image through environments. Never rebuild per environment.
- **Terraform remote state in GCS with locking**, separate state file per environment.
- **Keyless auth via Workload Identity Federation.**
- **Branch protection**: no direct pushes to `main`, required reviews, required status checks.
- **Conventional commits + semantic versioning** for release tags.
- **`terraform plan` on PR, `apply` only from `main`.** This one habit prevents most infrastructure incidents.


---

## 13. Cost governance (read this before you touch a console)

Your credit is a fixed resource and the failure mode is silent. Set the guardrails on day zero, in Phase 0, before any data service exists.

**Mandatory guardrails**

1. **Billing budget** on the project with alerts at 25%, 50%, 75%, 90% of your credit. Alerts go to your email.
2. **Programmatic kill switch:** budget alert → Pub/Sub → Cloud Function that disables billing on the project at 95%. Google documents this pattern. It is blunt and it will save you.
3. **BigQuery custom quota:** set a per-project *and* per-user daily query bytes limit (e.g. 50 GB/day). One accidental `SELECT *` on a badly partitioned table is the classic way people lose credits.
4. **Never write an unpartitioned fact table.** Require partition filters on large tables (`require_partition_filter = true`).
5. **GCS lifecycle rules** on every bucket from creation — landing to Nearline at 30 days, delete at 90.
6. **Table expiration** on all dev/stg datasets — 7 days. Terraform sets it; you cannot forget.
7. **Daily habit:** check the billing dashboard every morning for the first two weeks. Cost intuition is a skill and this is how you build it.

**The four things that actually burn trial credits, in order**

| Rank | Culprit | Control |
|---|---|---|
| 1 | Cloud Composer environment left running | Time-box it to Phase 7 only. Destroy it after. |
| 2 | Dataflow streaming job left running | `--maxNumWorkers=2`, drain it the same day. Never leave it overnight in Phase 4. |
| 3 | Persistent Dataproc cluster left running | Use Dataproc Serverless. If you create a cluster, set an auto-delete TTL at creation time. |
| 4 | Repeated full scans of a large unpartitioned table | Partition + cluster + custom quota |

Ingestion volume is almost never the problem. **Idle compute is.** Learn that lesson here rather than on your employer's bill — it is also the single most common cost finding in real GCP audits, so it is worth being the person who spots it.

**Keep data small deliberately.** Target: 2–5 GB total across the platform. BigQuery's free tier covers 10 GB storage and 1 TB of query processing per month. Design correctness does not require volume. If you want to *prove* scale handling, generate one large table once, run the partitioned vs unpartitioned query comparison, screenshot the bytes-scanned difference, and delete it.

---

## 14. Delivery phases

Each phase has a hard deliverable and an acceptance test. Do not move on until the acceptance test passes.

### Phase 0 — Foundation and guardrails (½ day)
**Build:** GCP project, billing budget + alerts + kill-switch function, enable APIs, GitHub repo, Terraform backend bucket with state locking, Workload Identity Federation between GitHub and GCP, service accounts, `terraform-plan-on-PR` workflow, resource labelling standard.
**Deliverable:** an empty but fully governed project where a PR produces a `terraform plan` comment.
**Acceptance:** you can create and destroy a test GCS bucket entirely through a PR merge, with zero console clicks and zero service account keys.
**Why first:** every hour spent here is repaid tenfold, and this is precisely the setup work you will be asked to do in your new role.

### Phase 1 — Storage foundation and data model (1–2 days)
**Build:** GCS zone buckets with lifecycle rules; BigQuery datasets for bronze/silver/gold/dq per environment; `dim_date` and `dim_time_of_day` generated; naming conventions documented; data contracts written as JSON Schema for all five sources.
**Deliverable:** the lakehouse skeleton, all in Terraform.
**Acceptance:** `terraform destroy` then `apply` recreates everything identically.

### Phase 2 — Batch ingestion: S3 files and S4 API (3–4 days)
**Build:** synthetic file generator; GCS-triggered Cloud Function validator with quarantine and duplicate detection; `bq load` path for clean CSV; Cloud Run Job API puller with retry, backfill mode, Secret Manager, schema-drift detection; `ingestion_audit` and `file_registry` tables.
**Deliverable:** two working ingestion paths landing in Bronze with full metadata.
**Acceptance:** drop a malformed file → it quarantines with a reason and alerts. Drop the same valid file twice → second is rejected as duplicate. Run the API job for a 7-day historical range → exactly one row set per day, rerunnable without duplication.

### Phase 3 — Batch ingestion: S5 reference data + Spark (2–3 days)
**Build:** Google Sheet external table, daily snapshot, SCD Type 2 merge into `silver_product`; Dataproc Serverless PySpark job parsing the messy nested JSON; a persistent Dataproc cluster created and destroyed once for comparison.
**Deliverable:** working SCD2 and a Spark job on GCP.
**Acceptance:** change a product's category in the Sheet, rerun, confirm the old row is closed with `valid_to` and a new current row exists — and that a historical query still returns the old hierarchy.

### Phase 4 — Streaming: S2 clickstream (4–5 days, the hardest phase)
**Build:** event generator on Cloud Run producing realistic sessions with deliberate duplicates, out-of-order events, and late arrivals; Pub/Sub topic + schema + dead-letter topic; Path A BigQuery subscription; Path B Beam pipeline with session windows, watermarks, allowed lateness, side output for late events, dedup, and enrichment; latency instrumentation.
**Deliverable:** both streaming paths, with measured p95 latency.
**Acceptance:** inject a duplicate event and an event 2 hours late; confirm the duplicate is dropped and the late event lands in the late-events table rather than corrupting the session aggregate. Then **drain the Dataflow job.**

### Phase 5 — CDC: S1 (3 days)
**Build:** Cloud SQL Postgres with private IP, seeded schema, mutation generator producing inserts/updates/**deletes**; Datastream stream to GCS/BQ; merge logic producing `silver_orders_current` and `silver_orders_history`.
**Deliverable:** working CDC with correct delete handling.
**Acceptance:** delete a row in Postgres; confirm it disappears from `_current` and is retained with a tombstone in `_history`. Apply a column addition in Postgres; confirm the pipeline handles the schema change without failing.

### Phase 6 — Transformation and quality (4–5 days)
**Build:** dbt project with the full Bronze→Silver→Gold DAG; incremental models on the large facts; all seven dimensions and six facts; comprehensive tests; reconciliation checks; one model built in Dataform for comparison; two models built as raw Airflow SQL operators for comparison; dbt docs published.
**Deliverable:** the complete dimensional model answering Q1–Q5.
**Acceptance:** `dbt build` runs green from a clean warehouse. Deliberately corrupt a source row that violates a business rule; confirm the correct test fails and halts the pipeline.

### Phase 7 — Orchestration and deployment (4–5 days)
**Build:** all DAGs — one per source, one per transformation domain — with dataset-based cross-DAG dependencies, dynamic task mapping for file fan-out, SLAs, alerting; local Docker development; self-hosted Airflow VM via Terraform; **then** Cloud Composer stood up via CI/CD, run 5–7 days, evidence captured, destroyed; Cloud Workflows for the ingestion trigger sub-flows.
**Deliverable:** the whole platform running unattended.
**Acceptance:** AC-1 — seven consecutive days of clean automated runs. And the same DAG bundle deployed successfully to both Airflow runtimes.

### Phase 8 — Serving, observability, and hardening (3–4 days)
**Build:** materialized views, BI Engine reservation (briefly), Looker Studio dashboard for Q1–Q5, reverse-ETL Cloud Run API for Q2, Dataplex registration with profiling scans, policy tags and masked views for PII columns, billing export dashboard, `INFORMATION_SCHEMA` query-cost analysis, full runbook, all ADRs written, README with architecture diagram.
**Deliverable:** a documented, observable, governed platform.
**Acceptance:** AC-5 — full destroy and rebuild in under 45 minutes, verified with a stopwatch. Hand the README to someone unfamiliar and have them explain the architecture back to you.

---

## 15. Risk register

| ID | Risk | Impact | Mitigation |
|---|---|---|---|
| R1 | Trial credits exhausted mid-build | Project halts | §13 guardrails; kill switch; time-boxed expensive services |
| R2 | Scope creep — "just one more service" | Never finishes | Phase gates with acceptance tests. Park extras in §18. |
| R3 | Beam/Dataflow learning curve stalls Phase 4 | Timeline slip | Path A (BQ subscription) delivers value independently; Path B can be time-boxed to 2 days and revisited |
| R4 | Job starts before project completes | Divided attention | Phases 0–2 alone already cover most of what a new project's first fortnight needs. Front-load those. |
| R5 | Free-tier quota limits on Datastream/Dataflow | Blocked | Check quotas in Phase 0; request increases early if needed |
| R6 | Over-engineering the model before data flows | Analysis paralysis | Build the thin vertical slice first: one source → Bronze → one Silver → one Gold table → one dashboard tile, end to end, before widening |

**R6 deserves emphasis.** After Phase 1, consider building one narrow end-to-end slice through *all* layers before returning to build each layer out fully. Vertical slice first, horizontal expansion second. It de-risks integration, and it means that at any point you have something demonstrable.

---

## 16. When they hand you the real SOW

Since you may be given a real one within days, here is what to do with it. This list is worth more to you next week than any GCP service.

**Read for these things, in order:**
1. **The acceptance criteria.** What, precisely, must be true for this to be "done"? If they are vague, that is the first thing you clarify — vague acceptance criteria are the single biggest cause of project overrun.
2. **The source inventory.** For each source: owner, format, volume, growth rate, latency requirement, refresh window, who to call when it breaks, and whether you have access *today*. Access requests take weeks in large organisations. Raise them on day one.
3. **The consumers.** Who queries the output, with what tool, at what concurrency, expecting what freshness? This determines your serving design more than anything upstream.
4. **Non-functional requirements.** Retention, PII/regulatory constraints, RPO/RTO, audit needs. These are usually buried and usually architecture-defining.
5. **What already exists.** Never design greenfield into a brownfield estate. Ask for the current architecture, the existing repos, the deployment conventions, the naming standards, the shared services you are expected to reuse.
6. **Dependencies on other teams.** Every one is a schedule risk. List them explicitly.

**Questions to ask in your first design conversation** (asking these makes you look senior, not junior):
- "What decisions will be made from this data, and how quickly do they need to be made?" — latency requirements should be derived, not assumed.
- "What's the cost envelope?" — architects who ignore cost get overruled by finance.
- "What's the expected data volume in 12 months, not today?"
- "Who owns each source system, and what's their change process?"
- "Is there an existing platform team, and what are their paved-road patterns?"
- "What does the on-call model look like once this is live?"

**On timelines:** when you are asked to estimate and you do not know, say "I can give you a confident estimate after a two-day spike on X." That is a professional answer. A number pulled from the air that you then miss is worse than a short, bounded delay.

---

## 17. A note on how to describe this work

Build this and you will have genuinely earned real skills — the design reasoning here is the same reasoning used on production systems, and the CI/CD, IaC, and governance practices are the real ones.

Describe it accurately, though: this is a self-directed platform build on synthetic data, not delivered production work for a client. Say exactly that, and lead with the reasoning rather than the tool list — "I built an end-to-end multi-source platform on GCP to work through the trade-offs between CDC and batch extraction, Dataflow and micro-batch, managed and self-hosted Airflow" is far stronger than an inflated claim, and it holds up under questioning. Inflated claims collapse in the first deep technical interview, and the collapse costs you more than the claim ever gained. You do not need the exaggeration — the work speaks.

One more thing, since you mentioned feeling behind: five years of Spark, SQL, Python, and Airflow is not a weak position. GCP is a vocabulary layer over concepts you already hold. You are not starting from zero; you are translating.

---

## 18. Deliberately deferred (the v2 backlog)

Naming what you chose *not* to build, and why, is part of a good design document. Candidates: Iceberg/BigLake tables for open-format storage; a feature store and a demand-forecasting model on Vertex AI; a semantic layer; data mesh domain separation; VPC Service Controls; CMEK; real-time alerting on anomaly detection; multi-region DR; data sharing via Analytics Hub.

