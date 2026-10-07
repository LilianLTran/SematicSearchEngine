# ADR 0004: Storage

Status: Accepted (2026-10-06)

## Context

The system must store up to about 500,000 article vectors together with metadata (PMID, title, abstract, year, journal), and answer nearest-neighbour queries over them. The project's central benchmark compares exact search (the ground truth) with approximate HNSW search, so the store must support both on the same data. Everything runs on one laptop in Docker, alongside other services (message broker, cache, monitoring), so simplicity and memory use matter.

## Options considered

1. **PostgreSQL with the pgvector extension.** One system holds vectors and metadata. Exact search works with no index (a full scan), and HNSW indexes can be added alongside. Standard SQL for filtering and upserts. Weaknesses: HNSW build time and memory grow quickly with data size, and there are fewer tuning and compression features than in purpose-built vector databases. **Chosen.**
2. **Qdrant (plus PostgreSQL for metadata).** Purpose-built, with strong filtering and quantization. Needs a second system and keeping metadata in sync. Deferred; possible later comparison.
3. **Milvus.** Designed for very large deployments, with several cooperating services. Too heavy for a single laptop. Rejected.

## Decision

Use PostgreSQL 16 with pgvector, run from the image `pgvector/pgvector:pg16` through Docker Compose, with a named volume for data and a health check.

- **Schema:** table `articles` with `pmid bigint primary key`, `title text`, `abstract text`, `year int` (nullable), `journal text`, `embedding vector(384)`.
- **Distance:** cosine distance, operator `<=>`, with the `vector_cosine_ops` operator class for indexes.
- **Exact baseline:** the table without a vector index.
- **Approximate search:** HNSW index. Starting parameters are the pgvector defaults (`m = 16`, `ef_construction = 64`, `hnsw.ef_search = 40`). Phase 4 sweeps these.
- **Writes:** upsert by PMID (`ON CONFLICT (pmid)`), so repeated loads are safe.

## Measurements

From `spikes/pg_spike.py` on 2,987 articles (MiniLM vectors, 384 dimensions):

| Measure | Result |
|---|---|
| Batched insert | 0.1 s |
| Exact search (no index) | median 4.53 ms, plan: sequential scan plus sort |
| HNSW search | median 0.47 ms, plan: index scan (index use forced with `enable_seqscan = off`) |
| Top-10 overlap, HNSW vs exact | 10/10 (one query only) |
| HNSW build time | 0.48 s |
| Table size (heap) | 4,440 kB (vectors stored inline; long text stored out of line and not counted) |
| HNSW index size | 5,984 kB (about 2 KB per vector) |

Query timings are database time only: they exclude the time to embed the query text. The same query was repeated 20 times, so caches were warm. pgvector extension version: 0.8.7 (the default version bundled with the `pgvector/pgvector:pg16` image, read from `pg_available_extensions`).

Rough linear extrapolation to 500,000 rows (to be verified):

- Vectors in the table: about 0.75 GB.
- HNSW index: about 1 GB.
- Exact search time: on the order of 0.5 to 1 second per query.
- HNSW build time is deliberately not extrapolated. It slows sharply when the graph no longer fits in `maintenance_work_mem` (default 64 MB), so it must be measured, with that setting raised to roughly 1 to 2 GB.

## Consequences

- Exact search cost grows linearly with row count, so HNSW is required to meet latency targets at scale, and exact search serves as ground truth for measuring recall.
- Filtered queries (for example by year) combined with HNSW can lower recall, because filtering may happen after the index returns candidates. If the API exposes filters, this needs testing.
- The database shares RAM with other containers, so the Docker memory limit and `maintenance_work_mem` must be set deliberately for the 500,000-row build.
- Storing vectors as 16-bit `halfvec` would roughly halve vector storage. It was not tested and is a possible later optimization.
- The credentials in the Compose file are for local development only and must move to environment variables before the repository is shared or infrastructure code is written.
- Revisit trigger: if the 500,000-row HNSW build cannot finish in about an hour with the available memory, or if query latency at the target recall is far above target, evaluate Qdrant and record the result in a new ADR.
