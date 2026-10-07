# ADR 0005: Ingestion

Status: Accepted as a design (2026-10-06). Runtime behaviour is to be verified in Phase 3; see "Not yet tested".

## Context

Up to about 500,000 PubMed articles must flow from files on disk, through embedding, into Postgres. The pipeline must survive worker crashes, tolerate duplicate deliveries, and handle bad records without stopping.

Facts already measured (ADR 0003 and ADR 0004):

- Embedding is the bottleneck: about 87 documents per second per process with MiniLM, so about 1.6 hours for 500,000 documents.
- A batched Postgres insert of about 3,000 rows takes about 0.1 seconds, so database writes are not a bottleneck.
- Embedding is CPU-bound and already uses several cores in a single process.

## Options considered

1. **Redpanda with plain Python consumers.** Redpanda is a Kafka-compatible broker that runs as a single binary with no JVM or ZooKeeper, which suits a laptop. Consumer groups provide partition-based parallelism, offset tracking, and replay, and plain Python keeps the logic visible. **Chosen.**
2. **Apache Kafka.** Same client API, but heavier to run locally. Rejected for local use.
3. **A task framework (Celery with a broker, or Ray).** Adds framework concepts on top of what consumer groups already provide. Rejected.
4. **A plain script with no broker.** The simplest way to load 500,000 rows once. A broker adds operational complexity that a one-off batch load would not need. It is chosen anyway because this project is meant to demonstrate a streaming ingestion architecture (backpressure, replay, failure isolation) and to support continuous ingestion of new articles.

## Decision

**Topics**

- `articles.raw`: 6 partitions, message key = PMID, JSON value containing pmid, title, abstract, year, journal. Retention capped by size (starting value 2 GB, since 500,000 messages is roughly 1 GB).
- `articles.dlq`: dead-letter topic for records that cannot be processed.

**Producer.** Reads the PubMed `.gz` files, applies the filters from ADR 0002, and publishes one message per article. It does no embedding.

**Consumers.** A consumer group named `embedder`. Each loop:

1. Poll up to about 128 messages, or until a short timeout (starting value 2 seconds).
2. Embed in batches of 16 (best throughput measured in ADR 0003).
3. Write all rows in one transaction using an upsert by PMID.
4. Commit the offsets only after the database transaction commits.

This is at-least-once delivery with idempotent writes: a message may be delivered twice, but writing the same article twice gives the same row.

**Error handling**

- *Temporary errors* (for example a database connection failure): retry up to 3 times with growing waits (1, 2, 4 seconds), then stop without committing offsets so the messages are re-delivered later. A database outage should stall the pipeline visibly, not push the whole stream into the dead-letter topic.
- *Permanent errors* (for example a record that fails validation or embedding): if a batch fails, process its records individually to isolate the bad one, then publish the original payload plus the error type, message, and timestamp to `articles.dlq` and continue.

**Client library.** `confluent-kafka` (to be confirmed to install on the project's pinned Python version).

## Measurements

No Redpanda measurements yet. The throughput figures above are inherited from ADR 0003 and ADR 0004.

## Not yet tested

- Whether a second consumer process improves throughput on one machine. The expectation is that it will not, because embedding is CPU-bound and one process already uses multiple cores.
- Redpanda's memory use under Docker Desktop's memory limit, alongside Postgres and the workers.
- The dead-letter path end to end, including a way to inspect and replay dead-lettered messages.
- Behaviour when a worker is stopped mid-batch: whether messages are re-delivered and rows are written exactly once.
- Whether the retry and timeout values above are reasonable. They are starting guesses.

## Consequences

- On a single machine, adding consumers will not speed up ingestion. The 6 partitions allow up to 6 parallel consumers, which only helps across several machines. On a laptop, plan for one or two consumers.
- Duplicate deliveries are possible and harmless because of the upsert.
- The dead-letter topic needs a small tool to inspect and replay its contents.
- Redpanda adds another container to the Compose stack. The Docker memory limit must cover Postgres, Redpanda, the workers, and later Redis, Prometheus, and Grafana.
- A broker is more machinery than a one-off load strictly needs. The README should say so openly and explain that the streaming design is the point of the project.
- Revisit trigger: if Redpanda proves unstable or too memory-hungry under the Docker limit, fall back to a simpler queue (Redis Streams or a Postgres-backed queue) and record the change in a new ADR that supersedes this one.
