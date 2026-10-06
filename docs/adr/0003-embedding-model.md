# ADR 0003: Embedding model

Status: Accepted (2026-10-05)

## Context

Each PubMed abstract is converted into a vector so that search can compare meaning rather than exact words. The system must embed up to about 500,000 abstracts and also embed every incoming search query.

Constraints:

- CPU only (Apple M4 MacBook, no GPU).
- Document embedding throughput limits how fast the corpus can be built and rebuilt. Re-embedding is expected several times during development.
- Query embedding sits on the request path, so its latency affects the cold-cache p95 target.
- The vector dimension affects storage size and index build time.

## Options considered

All measurements used fastembed (ONNX Runtime) on CPU, batch size 16, on abstracts from PubMed baseline file 1333.

1. **bge-small-en-v1.5, full length (up to 512 tokens).** 384 dimensions. Measured 19.3 docs/sec, so about 7.2 hours for 500,000 documents.
2. **bge-small-en-v1.5, input truncated to 1,000 characters.** Same model. Measured 42.2 docs/sec, so about 3.3 hours for 500,000 documents.
3. **all-MiniLM-L6-v2 (default input limit of about 256 tokens).** 384 dimensions, 6 layers instead of 12. Measured 87.6 docs/sec, so about 1.6 hours for 500,000 documents and about 10 minutes for 50,000. **Chosen.**

### Quality test

A label-free retrieval task on 2,987 documents: each abstract's own title is the query, and the target is that abstract ranked first. This is easier than real search, so it is useful only for relative comparison.

| Setup | recall@1 | recall@10 | MRR |
|---|---|---|---|
| MiniLM (default) | 0.986 | 0.999 | 0.992 |
| bge-small, truncated to 1,000 chars | 0.989 | 0.999 | 0.994 |
| bge-small, full length | 0.991 | 0.999 | 0.995 |

### Other measurements

- bge-small single-query embedding latency: median 3.9 ms, p95 4.2 ms (well below the cold-path target). MiniLM query latency was not measured separately; it is expected to be no slower because the model is smaller, and this should be confirmed.
- Throughput varies noticeably between runs. bge-small truncated was measured at both 27.7 and 42.2 docs/sec under slightly different settings, so all throughput figures should be read as rough, within about 40 percent.
- Setting the thread count manually (4 or 8) gave no clear benefit over the default in short runs.

## Decision

Use **all-MiniLM-L6-v2**, producing one 384-dimension vector per abstract (title and abstract concatenated), with no chunking. Access it only through the `Embedder` interface so the model can be replaced without touching service code.

## Consequences

- Document embedding is about 4 to 5 times faster than full-length bge-small, which keeps rebuilds cheap during development.
- The model only reads about the first 256 tokens, roughly 1,000 characters. The average abstract is 1,630 characters (ADR 0002), so the final third of a typical abstract, often the conclusions, is not represented in the vector. The truncated bge-small option has the same limitation.
- The quality test saturated: all three setups are within half a point on recall@1. It cannot show whether bge-small is better on harder, keyword-style queries. This is an open question, not a settled one.
- Both candidate models produce 384-dimension vectors, so switching later requires re-embedding the corpus but not a database schema change.
- Embedding is CPU-bound, so adding more consumer processes on one machine will not increase ingestion speed. This is recorded further in ADR 0005.
- Revisit trigger: if the Phase 4 evaluation set (harder queries, with exact-search ground truth) shows bge-small ahead by more than about 3 points of recall@10, switch to bge-small, truncated if time requires, and record the change in a new ADR that supersedes this one.
