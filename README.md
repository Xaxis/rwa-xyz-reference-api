# RWA.XYZ - Technical Proposal for Reference API Architecture

**Author:** Wil Neeley  
**Date:** 2025-08-11  
**Audience:** Charlie You - CTO, RWA.XYZ  
**Purpose:** Propose a robust, scalable, and high-quality **Reference API** that fits RWA.xyz’s current stack and supports company goals around consumer reach, schema agility, data quality, and breadth, while keeping metrics/analytics on their existing rails.

---

## 1. Industry Background - Reference Data Systems & APIs
Reference data is the slow-moving “truth layer” that lets every downstream system agree on what an asset/issuer/platform/network **is**, independent of volatile prices or transactions; a good Reference API wraps that truth in stable contracts so external and internal consumers can search, resolve identifiers, sync changes, and judge trust using provenance and quality signals.

**Key points from mature reference APIs**
- Canonical identifiers per entity; comprehensive mappings to chain addresses and external IDs.  
- Stable versioned contracts (`/v1`) with additive evolution; deprecations, not breakages  
- Low-latency read paths (edge + hot cache) with DB as authoritative fallback.  
- Incremental sync for consumers via change feeds, webhooks, and snapshots  
- Governance surfaced in payloads: lineage, curation, and data-quality status.

---

## 2. Current RWA.XYZ Architecture Assessment
RWA.xyz already has a strong ingestion → lakehouse → serving pattern; the Reference API should plug cleanly into this without disturbing metrics/analytics and without overloading the operational store.

**Data ingestion (what + why)**
- Blockchain events via **Delta Sharing** to avoid bespoke indexing and enable zero-copy vendor feeds.  
- **Partners App** (Next.js+tRPC+Postgres) and **Input App** (Next.js+tRPC) to curate and approve first-party data before it becomes canonical.  
- **Partners API** (FastAPI+Postgres) for machine-to-machine metadata at scale; **daily metrics to GCS** to keep time-series off the reference path.  
- Targeted external API jobs for gap-filling where vendors/partners don’t cover.

**Processing in the lakehouse (what + why)**
- **Databricks on GCP** with **Delta Lake** and **Medallion** (Bronze→Silver→Gold) to enforce quality stages.  
- **Workflows** for orchestration, **Monte Carlo** for DQ, and **row-level lineage** so every field is traceable.

**Serving stores and APIs (what + why)**
- **Reference DB (Cloud SQL Postgres)** as operational “reference master” for entities and lookups.  
- **DWH (Cloud SQL Postgres + DBT)** optimized for analytics and the app  
- **Main API** (Next.js/Vercel + Cloudflare) serves app + analytics queries;  
- **Reference API** surfaces canonical entity metadata from the Reference DB (today a thin, DB-backed layer).

**Strengths**
- Mature ingest + transformation with DQ and lineage.  
- Clean separation of operational vs analytical stores.

**Gaps**
- Read-heavy traffic can stress Postgres.  
- No consumer-facing canonical ID map/resolve surface.  
- No first-class change feed/webhooks; partners over-poll.  
- Search/discovery runs on the operational DB.  
- Limited schema agility for new asset families.  
- Lineage/DQ not exposed to consumers.

---

## 3. Pain Points & Risks
If we don’t optimize the serving path, external reads will compete with ingestion/curation without canonical mappings, joins are brittle without change feeds and webhooks, partners waste bandwidth and miss SLAs, operational DB search will trend slow at scale, and trust suffers when consumers can’t see provenance or quality.

**Top risks to address**
- DB saturation from reads (long term) 
- Identifier mismatch/duplication  
- Inefficient sync patterns  
- Search latency on mixed filters  
- Invisible lineage/quality

---

## 4. Proposed Reference API Architecture
The design is intentionally **stack-true**: Python on Cloud Run, GCP managed services, Cloudflare at the edge, DBT for serving prep, Monte Carlo for DQ, so we add capability without introducing unnecessary moving parts.

### Goals (what & why)
- **Scalable:** move reads to edge/hot cache first to protect Postgres and stabilize latency.  
- **Trustworthy:** make lineage and quality visible so integrators can self-assess fitness-for-use.  
- **Flexible:** typed cores plus JSON extensions to add breadth without breakage  
- **Consumer-first:** identifier resolution, clean search, and efficient sync (changes/webhooks/snapshots).

### 4.1 High-Level Design (what happens where)
- **Ingestion & processing (unchanged):** keep Medallion pipelines, Monte Carlo, and lineage; Gold exports refresh **Reference DB**; metrics stay **GCS → metrics pipeline → DWH**.  
- **Serving tier (new):** **Cloudflare Workers** (edge cache) → **Memorystore (Redis)** (hot ID lookups) → **Cloud SQL Postgres** (authoritative fallback + Phase-1 search); precise invalidation keeps caches coherent.  
- **Reference API service:** **FastAPI on Cloud Run** (aligns with Partners API) using Pydantic models and OpenAPI for SDKs; the API is primarily read, leaving writes with Partners/Input apps.  
- **Change notifications:** **Datastream (or logical decoding) → Pub/Sub** emits entity changes; an invalidation worker evicts Redis, purges Cloudflare, and (if added later) reindexes search.

### 4.2 Key Components (what & why)
- **Canonical ID registry:** stable, opaque IDs (`asset_id`, `issuer_id`, `platform_id`, `network_id`) plus `identifier_maps` for chain addresses, partner IDs, registry IDs, and slugs—so every foreign identifier resolves cleanly.  
- **Read path optimization:** edge cache for public GETs, Redis for hot entities, Postgres only for misses/`as_of`; **materialized views**, **FTS (`to_tsvector`, `pg_trgm`)**, and **JSONB GIN** keep fallback fast without new infra.  
- **CDC & sync:** `/v1/changes?since=…` as an ordered feed, **webhooks** (HMAC-signed) for push, and **GCS snapshots** (NDJSON/Parquet) for bulk loads—so partners stop re-pulling everything.  
- **Schema agility (extensions):** small typed cores with namespaced `extensions` (JSONB) per asset family, validated with versioned JSON Schemas—so new attributes ship quickly without breaking `/v1`.  
- **Data quality & provenance:** optional includes return `lineage_id`, `source_system`, `ingestion_ts`, `curation.by/at`, and a minimal `quality {status, score}`, so consumers can filter by quality or inspect provenance when needed.

---

## 5. Caching & Performance
We target predictable, low latency by caching at the edge and in Redis, and by shaping the DB for fast fallback; coherence is guaranteed by event-driven purges and not guesswork.

**Plan**
- **Cloudflare Workers:** cache public GETs, short TTLs (5–15m) for semi-dynamic and 24h for static, `stale-while-revalidate`, purge by surrogate keys (e.g., `asset:ast_123`).  
- **Memorystore (Redis):** compressed entity summaries keyed by canonical ID; TTL 6–24h; lazy rehydrate on miss.  
- **Cloud SQL Postgres:** materialized views for hot joins; FTS + JSONB GIN; `as_of` hits versioned tables.  
- **SLO targets:** cache-hit p50 ≈ ≤75 ms, p95 ≤250 ms; availability ≥99.9%.  
- **Phase-2 search (optional):** add Elastic/OpenSearch behind the same endpoints only if query mix demands.

---

## 6. Security & Access Control
Security should be boring, predictable, and aligned with existing controls so partners integrate once and focus on data, not auth trivia.

**Approach**
- **AuthN:** OAuth2 client credentials for partners; short-lived JWTs; small whitelisted public reads behind Cloudflare rate limits.  
- **AuthZ:** scoped tokens (`reference.read`, `reference.search`, `dq.read`).  
- **Webhooks:** HMAC signatures + replay protection; optional mTLS for premium partners.  
- **Audit & data hygiene:** admin merges/edits fully audited; keep reference payloads non-PII and strip/redact at ingest if necessary.

---

## 7. Operational Considerations
We’ll make the service observable from day one, roll out safely, and keep costs in check by emphasizing cache hits and precomputation where it pays off.

**Ops plan**
- **Observability:** OpenTelemetry traces; Prometheus/Grafana RED dashboards; Sentry for errors.  
- **Safety:** Cloud Run canaries; feature flags for new fields; contract tests on OpenAPI in CI.  
- **Cost control:** monitor cache hit rates; DBT materializations for heavy joins; tune Cloud Run concurrency/autoscaling.

---

## 8. Implementation Roadmap
The path prioritizes low-risk wins first (read path + sync), then trust surfacing, and only introduces heavier search infra if truly warranted.

**Phases**
- **Phase 1 - Foundations:** canonical IDs + `identifier_maps`; FastAPI on Cloud Run; Cloudflare + Redis; GET/list/resolve/ids; Postgres FTS search; `/changes` via Datastream→Pub/Sub.  
- **Phase 2 - Sync & Trust:** webhooks with retries; daily GCS snapshots; lineage/versions endpoints; quality/provenance includes; `/dq/events`.  
- **Phase 3 - Search scale (optional):** Elastic/OpenSearch; reindex from Pub/Sub; API contract unchanged.  
- **Phase 4 - SLAs & Ops:** load tests; SLO dashboards/alerts; autoscaling tuning.

---

## 9. Benefits to RWA.XYZ
This plan shields Postgres from read spikes, exposes the trust signals partners need, gives clean primitives for integration (resolve/IDs, filters, search, changes, webhooks, snapshots), and lets you expand schema breadth quickly via extensions, while staying faithful to your GCP/Databricks/Cloudflare stack and leaving metrics/analytics on their proven pipeline.

---

## 10. Closing Statement
A focused Reference API built this way becomes the fast, coherent front door to RWA.xyz’s canonical facts: easy to read, easy to sync, and honest about provenance, so external consumers and internal teams can move faster without re-implementing plumbing or guessing at data trust, and the platform scales consumer reach, schema agility, quality, and breadth with minimal operational risk.
