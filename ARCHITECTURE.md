# Margio Architecture

A deeper look at how Margio is built. The design goal is simple: resolve the easy, unambiguous majority of items with cheap deterministic logic, and only spend fuzzy or AI effort on what is genuinely uncertain, while never letting one wrong match silently corrupt a real quote.

---

## The matching pipeline

An estimate item flows through ordered layers. Each layer either resolves the item confidently and stops, or passes candidates down with a score.

### Layer 0: Normalization
Parses the incoming price list (CSV / XLSX) and the estimate, cleans identifiers (EAN, SKU), and unifies units. This is where messy real-world input becomes something the rest of the pipeline can reason about. Column mapping is automatic. The app detects which column is which even though every supplier exports a different format.

### Layer 1: Exact Match
Tries hard identifiers in order of trust: EAN first, then SKU, then the company's own dictionary, then normalized product name. Most real-world items resolve here, instantly and unambiguously. The company dictionary is key. It is populated by the human-in-the-loop step, so each tenant's exact-match hit rate climbs over time.

### Layer 2: Parameter Extraction
Pulls technical parameters (for example cable cross-sections and pipe diameters) from product descriptions, with no AI, pure parsing. This is what stops a semantically similar but technically wrong match. A `3×2.5mm²` cable and a `3×1.5mm²` cable look almost identical to a text-similarity model, but they are different products at different prices. Parameters are extracted and used to constrain matching before any fuzzy step runs.

### Layer 3: Semantic Search
For items still unresolved, catalog products are embedded and searched with pgvector similarity. Crucially, semantic similarity only chooses within the same parameter class established in Layer 2. It breaks ties, it does not override hard technical facts.

### Layer 4: LLM Re-ranking (planned)
For medium-confidence items, an LLM picks the best candidate from the top-k. It is designed as an **optional, cost-bounded** layer: it only fires on the narrow medium-confidence band, sends just the top few candidates (never prices), has a hard token budget per project, and falls back to human review when disabled. Because accepted decisions are cached to the company dictionary, the same item never hits the model twice. Not yet enabled. The system currently routes medium and low confidence straight to human review.

### Layer 5: Human-in-the-Loop
Low-confidence items surface to the user for a manual decision. That decision is written back into the company dictionary, so the identical item resolves automatically at Layer 1 next time. The system gets sharper per tenant with use.

---

## Confidence Aggregator

The router at the center of the pipeline. It scores each item's candidates and decides the path:

```
high confidence   → AUTO_ACCEPT
medium confidence → LLM re-ranking (planned) or human review today
low confidence    → Human-in-the-Loop → company dictionary
```

Thresholds live in one config module (`lib/config/thresholds.ts`) rather than being scattered as magic numbers, so routing behavior is tunable in one place.

---

## Multi-tenancy & security

- **Every** data-access function takes `tenantId` and every query filters on it. There is no code path that reads company data without a tenant filter.
- `tenantId` is read **only from the verified JWT session**, never from the request body, so a client cannot request another tenant's data by changing a payload.
- Data access is **raw parameterized SQL** (`$1, $2, ...`) through `node-postgres`, with no string interpolation and no ORM. This keeps queries explicit, injection-safe, and easy to reason about for a multi-tenant model.
- Uploaded files (untrusted input) are validated and handled in a **file sandbox** (type and size checks) before parsing.

---

## Data & background work

- **PostgreSQL 17 on Supabase** with `pgvector` (semantic search) and `pg_trgm` (fast fuzzy text lookup).
- Schema evolves through **versioned SQL migrations**. pgvector, price-list deduplication, trigram indexing, column-mapping storage, and project snapshots each landed as a migration.
- **Snapshots and diffs** let a quote be frozen and later compared against live prices, which is what powers the margin-alert feature.
- Heavy ingestion (parsing large price lists, URL sync) runs through a **background job processor** and cron routes, not inside the request and response cycle.

---

## Design principles (enforced throughout)

- TypeScript **strict**, no `any`.
- One responsibility per function, error handling on every async operation.
- No magic numbers. Constants live in `lib/config/`.
- Parameterized SQL only.
- Tenant isolation is non-negotiable and checked at every query.

---

## Roadmap

- **Phase 3:** enable the optional LLM re-ranking layer for the medium-confidence band, with prices withheld from the model and a hard token budget per project.
- Stripe billing.
- Broader supplier URL connectors and scheduled sync.
