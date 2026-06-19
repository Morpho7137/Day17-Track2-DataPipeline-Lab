# Vietnam Returns Intelligence Pipeline

## Problem
I would build a data pipeline for a Vietnamese e-commerce marketplace that needs to understand returns, refunds, and shipping disputes across orders, chat transcripts, support notes, and policy documents. The business problem is not just storage. It is to answer operational questions quickly and safely: which orders are eligible for return, which refund claims are suspicious, which warehouses create the most exceptions, and which policy clause applies when an agent escalates a case.

The inputs are messy. Orders arrive from OLTP exports, support tickets arrive as free text, some documents are PDFs with mixed Vietnamese and English, and some evidence comes from screenshots or scanned forms. Vietnamese text matters because accented characters are easy to mangle, names can be ambiguous without diacritics, and OCR errors are common. The system also has hard cost and failure constraints: the team wants local DuckDB-style processing for most workflows, minimal API dependence, and a design that can survive replays without duplicating refund cases.

## Key Decisions

### 1. Batch or streaming
I would make the core pipeline batch-first and add streaming only for the few event types that truly need freshness, such as support ticket status changes or refund approvals.

Tradeoff: streaming gives faster visibility, but it increases operational load, replay complexity, and debugging cost. Batch gives a simpler failure model and is enough for order-level analytics, policy curation, and weekly model refreshes. I would choose batch for the main warehouse, then use a small stream processor for live ticket updates because that is where latency changes the user experience.

### 2. Contracts and quarantine
Every source gets a schema contract. Bad rows go to quarantine, not to repair-by-default.

Tradeoff: automatic repair increases throughput, but it also hides upstream regressions. For Vietnamese addresses and support text, I would allow only safe normalization such as trimming whitespace, standardizing timestamps, and parsing known date formats. I would not silently infer product identity or refund eligibility. If quarantine spikes, the pipeline should alert instead of guessing.

### 3. Train/serve parity
Features for fraud or return-risk models must be point-in-time correct. I would use an ASOF-style join for user history, warehouse history, and prior refunds.

Tradeoff: the latest-value join is easier to write, but it leaks future information and creates models that look good offline and fail in production. The safer design is slightly harder to implement because every feature table needs timestamps and event ordering, but that is mandatory here. Refund propensity is exactly the kind of problem where a future approval or a later complaint can contaminate the label row.

### 4. Vector search or knowledge graph
I would use vector retrieval for policy lookup and support-answer grounding, and a knowledge graph for multi-hop operational questions.

Tradeoff: vectors are better for fuzzy language, paraphrases, and Vietnamese OCR noise. Graphs are better when the answer depends on relationships across orders, warehouses, policies, and claims. For example, "which warehouses handled returnable widgets that shipped from Hanoi and were refunded after a policy exception" is a graph question. A flat chunk retriever is fine for "what is the 30-day return policy". I would not force one tool to do both jobs.

### 5. Idempotency and backfills
Every ingest step should be idempotent by source event id or natural key. Backfills should be partitioned by business date and rerunnable.

Tradeoff: append-only writes are simple, but they can double-count on reruns. Overwriting partitions is safer for backfills, but it must be scoped carefully so one day of repair does not destroy unrelated data. My default would be partition overwrite for curated tables and append-only for raw bronze tables.

## Rejected Alternative
The main alternative I would reject is an all-vector design: dump tickets, policies, and order notes into embeddings and answer everything with similarity search.

Why reject it: it works for rough text lookup, but it fails on provenance, multi-hop relationships, and policy conflict resolution. It also makes auditing hard. When a refund is denied, the team needs to know exactly which policy clause, order event, and agent action led to the decision. A graph plus structured tables gives that traceability; vectors alone do not.

## Architecture Sketch

```text
            +------------------+
Orders ----> | Bronze ingest    | ----+
Tickets ---> | raw append-only  |     |
Docs ------> | OCR / cleanup    |     |
            +------------------+      |
                     |                |
                     v                v
            +------------------+  +------------------+
            | Validate / gate  |  | Trace capture    |
            | quarantine bad   |  | support events   |
            +------------------+  +------------------+
                     |                |
                     v                v
            +------------------+  +------------------+
            | Silver / typed   |  | Eval + training  |
            | dedup + PIT      |  | dataset curation |
            +------------------+  +------------------+
                     |                |
          +----------+--------+       +------------------+
          |                   |       |                  |
          v                   v       v                  v
   +-------------+     +-------------+  +-------------+  +-------------+
   | Gold tables  |     | Feature set |  | Vector RAG  |  | Knowledge   |
   | reporting    |     | ASOF joins  |  | policy text |  | graph       |
   +-------------+     +-------------+  +-------------+  +-------------+
```

## Why this fits Vietnam
Vietnamese text handling is not an edge case here. It affects every stage: OCR quality, matching customer names, search recall, and support review. I would budget for normalization of accents, address aliases, and common OCR mistakes. I would also be conservative on external API use because bandwidth and vendor cost matter more when the corpus is large and the corpus is not perfectly clean.

The result is a pipeline that is cheap to rerun, safe to backfill, and honest about uncertainty. It does not try to be fancy everywhere. It uses structure where provenance matters, vectors where language is fuzzy, and graphs where the answer truly spans multiple facts.
