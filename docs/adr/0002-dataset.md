# ADR 0002: Dataset

Status: Accepted (2026-10-05)

## Context

The semantic search engine needs a realistic, free text corpus of at least 500,000 documents, with enough variety in topic and length to make search quality and latency measurements meaningful. The corpus must be available in bulk, so it can be streamed through the ingestion pipeline repeatedly without depending on a rate-limited API.

## Options considered

1. **PubMed baseline files (abstracts).** Bulk XML downloads from NCBI, about 30,000 records per file, covering a very wide range of biomedical topics. Abstracts are self-contained, around 1,600 characters on average, and need no chunking. **Chosen.**
2. **SEC EDGAR filings.** Rich text, but requires rate limiting (identifying User-Agent, roughly 10 requests per second), messy HTML parsing, and very long documents that would need chunking. Rejected for the first version; possible later extension.
3. **Both sources.** More realistic variety, but roughly doubles the ingestion and parsing work before any search quality has been measured. Rejected for now.

## Decision

Use PubMed baseline files, starting from the highest-numbered files and working backwards.

- **Filters:** keep only records whose language is English and whose abstract is non-empty. Also drop abstracts shorter than 200 characters, because the sample showed some one-sentence abstracts.
- **Fields kept:** PMID, title, abstract, year, journal. Year can be null and the schema must allow it.
- **Scale plan:** start with 50,000 records (2 files), then scale to 500,000 (about 20 to 21 files).
- **Parsing:** stream from the `.gz` files with `lxml.iterparse` and clear elements as they are processed. Files are never extracted to disk.

## Measurements

Measured with `spikes/parse_pubmed.py`:

| Measure | File 1333 (full) | File 1334 (partial) |
|---|---|---|
| Total records | 30,000 | 4,989 |
| English | 29,589 (98.6%) | 4,889 (98.0%) |
| English with abstract | 26,081 (86.9%) | 4,030 (80.8%) |
| Average abstract length | 1,630 characters | 1,610 characters |

Derived planning figures, using the full file:

- 50,000 usable records needs about 2 files.
- 500,000 usable records needs about 20 files, so plan for 20 to 21 to leave room for the minimum-length filter.

Two findings from the measurements:

- **The highest-numbered file is partial.** It holds only the records added since the last full file (4,989 records versus 30,000). Planning from it alone would have overestimated the number of files needed by about six times. Always check the record count of a file before extrapolating.
- **Newest files are not guaranteed to be dense in abstracts**, but both files tested were above 80%. An older low-numbered file (`pubmed26n0010`) was downloaded but never parsed, so the "newest first" choice is not yet backed by a direct old-versus-new comparison.

## Consequences

- Abstracts need no chunking, so one vector per document is enough (see ADR 0003).
- Ingestion must handle null years and very short abstracts.
- The data is a biomedical corpus only. Search quality results will not automatically generalize to other domains, and the README should say so.
- Data files stay out of the repository, both because of size and because NLM's terms of use should be respected. The repository contains download and parse scripts, not data.
- If time allows, EDGAR can be added later as a second source to test a different text type.
