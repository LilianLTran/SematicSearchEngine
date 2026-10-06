# ADR 0001: Record architecture decisions

Status: Accepted (2026-10-05)

## Context

This project involves many technical choices: which dataset to use, which embedding model, where to store vectors, how to ingest data, and how to measure success. Each choice has trade-offs, and it is easy to forget the reasoning behind a decision a few weeks later. This document is also used as a reference for my future decision.

## Options considered

1. **Keep no record.** Fastest at the start, but the reasoning is lost and the design is hard to defend or revisit later.
2. **Keep notes in the README.** Everything is in one place, but the README becomes long and cluttered, and old decisions get overwritten instead of preserved.
3. **Write one short file per decision.** Each decision is small, dated, and easy to find. Old decisions can be marked as superseded rather than deleted.

## Decision

Use one short Architecture Decision Record (ADR) file per significant decision, stored in `docs/adr/`. Files are numbered in order (for example `0002-dataset.md`) and follow the same four headings: Context, Options considered, Decision, Consequences. If a decision changes later, the old ADR is marked as superseded and a new one is added.

## Consequences

- A little extra writing for each decision.
- The reasoning behind the design is preserved and reviewable.
- Changes of mind become visible and documented, instead of silently overwriting history.
- The ADRs double as portfolio material that shows engineering judgment.
