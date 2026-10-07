# ADR 0006: Success targets

Status: Accepted as hypotheses (2026-10-06). Every number below is a starting target, to be revised after real benchmarks in Phase 4 and Phase 5.

## Context

The project needs measurable goals, so that "fast" and "good" can be checked rather than claimed. The original project blueprint asks for "multi-thousand RPS" throughput and p95/p99 latency under concurrent load, but gives no hardware, query mix, or quality bar. The system will be measured on one Apple M4 MacBook, with the load generator (Locust) sharing the machine with the system under test. RAM and the Docker memory limit are to be recorded before benchmarking.

Two query paths matter, and they have very different costs:

- **Cold path:** a query not seen before. The text must be embedded, then searched in Postgres.
- **Warm path:** a query whose result is already cached in Redis.

## Options considered

1. **Adopt the blueprint's aspirational numbers as they stand.** Simple, but the numbers have no hardware context and no basis in measurement. Rejected.
2. **Set targets from measured inputs and label them as hypotheses.** Targets are grounded in numbers already measured, and revised openly when real benchmarks arrive. **Chosen.**
3. **Set no targets until benchmarking is done.** Nothing to design or test against, and results cannot be judged. Rejected.

## Decision

### Metric definitions

- **recall@10:** for each query in an evaluation set, the fraction of the true top 10 results (from exact search with cosine distance) that the HNSW search also returns, averaged over all queries. The evaluation set will contain at least 300 queries.
- **p95 latency:** the time within which 95 percent of requests complete, measured at the API, in steady state.
- **Cold cache:** the Redis cache is empty for that query. **Warm cache:** the identical query was served before and its result is in Redis.
- **Sustained throughput:** the highest request rate at which the latency target and the error target for that path both still hold. A rate that violates either does not count.

### Targets (hypotheses)

| Area | Metric | Target |
|---|---|---|
| Quality | recall@10 vs exact search, 500,000 rows, chosen HNSW setting | at least 0.95 |
| Latency | warm-cache p95, 50 concurrent users | under 50 ms |
| Latency | cold-cache p95, 50 concurrent users | under 200 ms |
| Throughput | warm cache, sustained | at least 1,000 requests/sec (stretch: 2,000) |
| Throughput | cold cache, sustained | at least 100 requests/sec |
| Reliability | server errors (5xx) during sustained load | under 0.1% |
| Ingestion | 500,000 documents end to end | at most 2.5 hours; under 0.5% dead-lettered |
| Index | HNSW build at 500,000 rows | at most 1 hour |
| Engineering | test coverage, strict typing, lint | coverage at least 85%, mypy strict and ruff passing in CI |
| Engineering | reproducibility | one command, `docker compose up --build`, starts all services |

### Measurement method

- Load generator: Locust, with 1 minute of warm-up followed by 5 minutes of steady state at each concurrency level (10, 50, 100, 200 users).
- Cold-path test uses a large set of distinct queries. Warm-path test uses a smaller set with repeats, and the cache hit rate is reported.
- Latency percentiles (p50, p95, p99) and request and error rates come from the Prometheus metrics and are cross-checked against Locust's own report.
- Both cold and warm results are always reported separately, together with the hardware, the dataset size, and the HNSW parameters.

## Measurements (inputs to these targets)

- Query embedding with bge-small: median 3.9 ms, p95 4.2 ms (ADR 0003). The chosen MiniLM model was not measured separately and is expected to be no slower.
- Exact search: median 4.53 ms and HNSW search: median 0.47 ms, on about 3,000 rows (ADR 0004). These are database time only. HNSW use was forced for the comparison, and nothing has been measured yet at 500,000 rows.
- Embedding throughput: about 87 documents per second per process with MiniLM, so about 1.6 hours for 500,000 documents (ADR 0003).
- Not yet measured: API overhead, Redis lookup time, HNSW search time at 500,000 rows, and throughput under concurrent load.

## Consequences

- The cold-path target of 200 ms rests on an unmeasured HNSW search time at 500,000 rows. If that search takes much longer than expected, the target will be missed or revised, and the result recorded either way.
- Locust shares CPU with the system under test, so measured throughput will be lower than the system could deliver on dedicated hardware. This limitation must be stated alongside every throughput number.
- The warm-path throughput target depends mostly on API and Redis overhead, not on search. It is the least certain of the throughput targets.
- "Multi-thousand RPS" is realistic only for cached queries. Cold-path throughput is limited mainly by embedding and search, and is expected to be in the low hundreds.
- Changes to a target are recorded as dated entries in this file, keeping the original numbers, so the difference between expectation and result stays visible. A large change gets its own ADR.
- If a target is missed, the shortfall and the reason are reported honestly in the README benchmark section. A missed target with a clear explanation is better evidence than a hidden one.
