# Margio Architecture

A deeper look at how Margio is built. The design goal is simple: resolve the easy, unambiguous majority of items with cheap deterministic logic, and only spend fuzzy or AI effort on what is genuinely uncertain, while never letting one wrong match silently corrupt a real quote.

---

## The matching pipeline

An estimate item flows through ordered layers. Each layer either resolves the item confidently and stops, or passes candidates down with a score. All of it runs through one side-effect-free function, `decideMatch()`: it reads but never writes, so the production route and the evaluation script execute exactly the same code, and the route only persists the outcome.

### Layer 0: Normalization
- **Price lists and the estimate** are parsed from CSV / XLSX.
- **Identifiers** (EAN, SKU) are cleaned and units are unified.
- **Column mapping is automatic.** The app detects which column is which even though every supplier exports a different format.
- **Numbers are parsed by locale.** The last separator is the decimal one, so `1.234,56` is not read as `1.234`.
- **The decimal comma inside a product name survives.** `3x2,5` stays a 2.5 mm² cable instead of collapsing into `3x25`, which would be a different product. A data migration rewrote names and dictionary keys stored the old way.

### Layer 1: Exact Match
- **Hard identifiers come first, in order of trust:** EAN, then SKU, then the company's own dictionary, then the normalized product name. Most real-world items resolve here, instantly and unambiguously.
- **The company dictionary** is populated by the human-in-the-loop step, so each tenant's hit rate climbs over time.
- **A human-sourced entry is not trusted blindly.** It is used only when its wording is close to the chosen product (pg_trgm similarity ≥ 0.5) or the same target was confirmed at least twice. Otherwise the item falls through to the later layers.

### Layer 2: Parameter Extraction
- **What it does:** pulls technical parameters from product descriptions with plain parsing, no AI. It covers:
  - cable cross-sections and conductor size;
  - diameters (`fi25`, `ø25`, `20mm`) and volume (`18 l` vs `35 l`);
  - breaker curve and rating (`B16`, `C20`), residual current (`30mA`) and IP class;
  - power (`40W`, `2x36W`), voltage and lamp base (`E27`);
  - insulation colour.
- **The rule:** a candidate is dropped only when both names state the parameter and the values differ.
- **Dimensions vs cross-sections:** `NxM` counts as a cable cross-section only when M is a standard conductor size. So `600x600` and `60x60cm` are recognised as the same panel size, not as a cable.
- **Why it matters:** a `3×2.5mm²` cable and a `3×1.5mm²` cable look almost identical to a text-similarity model, but they are different products at different prices.
- **Where it applies:** to the Layer 1 hit and to every semantic candidate, before any fuzzy step can choose.

### Layer 3: Semantic Search
- **What it does:** for items still unresolved, catalog products are embedded (OpenAI) and searched with pgvector.
- **Where its authority ends:** semantic similarity only chooses among candidates that passed Layer 2. It breaks ties, it never overrides hard technical facts.
- **What it skips:** archived and superseded price lists. When several entries qualify, the newest valid one wins, so re-running a match gives stable results.

### Layer 4: LLM Re-ranking (optional, off by default)
- **When it runs:**
  - when the best Layer 3 candidate sits in the ambiguous similarity band;
  - to confirm the matches Layer 3 would auto-accept (similarity ≥ 0.9). The measurement showed this is where the remaining errors hid. If the LLM does not answer on a confirmation, the item goes to human review, never to an auto-accept.
- **What the model sees:** the estimate line and up to 5 candidates that survived Layer 2, each with a name and a unit. It never sees prices, and the candidate type has no price field, so the type system enforces this.
- **What the model can answer:** a strict JSON schema whose `choice` is an enum of the shown candidate numbers or `null`, plus a 0–100 confidence and a short reason. It cannot invent an option or echo a malformed ID.
- **What happens next:**
  - High confidence: auto-accept, recorded as Layer 4.
  - Anything else: human review, with the LLM's pick stored on the item and shown in the review queue as an "AI suggestion".
  - Errors, timeouts or an invalid answer: human review without a suggestion.
- **Model:** a non-reasoning model at temperature 0. Reasoning model families, which reject the temperature parameter, are handled.
- **Switch:** turned on with an environment variable, no code change.

### Unit gate
- **The problem:** a correct product sold in a different unit still prices a quote wrongly. Think of a line in metres matched to a price per 100 m drum.
- **Pack sizes from names:** they are read from product names (`krążek 100m`, `op. 100 szt`) and turned into a conversion factor stored on the item.
- **No conversion, no auto-accept:** an auto-accept whose units cannot be converted is downgraded to human review. The factor stays editable by the user.

### Layer 5: Human-in-the-Loop
- **Where items surface:** low-confidence and gated items go to the user for a manual decision.
- **What a decision does:** it is written into the company dictionary with a confirmation count, so the identical item resolves at Layer 1 next time, once it is trusted.
- **Undoing mistakes:** the dictionary is visible in settings, with source, confirmations and trust status, and entries can be deleted.

---

## Measuring the LLM layer

The evaluation script runs `decideMatch()` over labelled sets of estimate-style lines. The first set has 57 lines:

| Group | Lines |
|---|---|
| Ordinary paraphrases | 26 |
| Tricky lines that do have a correct match | 6 |
| Tricky lines with no correct match (`B16` vs `B20`, `IP44` vs `IP54`, 12 vs 24 modules) | 18 |
| Products not in the catalog | 7 |

- **What counts as a hit:** the same product from a different wholesaler.
- **Two runs:** one with Layer 4 off (the baseline) and one with it on. Layer 1–3 results and embeddings are shared between them.
- **What it reports:**
  - top-1 accuracy;
  - false auto-accepts on lines that should have no match;
  - auto-accept rate and precision;
  - cost and latency (mean and p95);
  - how items split across layers;
  - a similarity histogram;
  - a confidence-threshold sweep.
- **Dry run:** a `--dry-run` mode shows how many items would reach the LLM without calling it. A `--model` flag compares models.
- **Safety:** the script is read-only against the database.

### Results

**First run (57 lines):**
- The LLM raised the auto-match rate from 25% to 58%, and its own decisions were right in 18 of 19 cases.
- 6 wrong auto-accepts remained. All came from Layer 3 (similarity ≥ 0.9), which the LLM did not see.
- Fix: Layer 2 was extended to catch those traps.

**Held-out set (39 lines).** The Layer 2 rules were written from the first set's errors, so re-measuring on it would be inflated. A second set with new wording, new traps and a second real supplier price list measures generalisation.

| `gpt-4.1-mini`, threshold 85 | Without LLM | LLM in the ambiguous band | + LLM confirms ≥ 0.9 |
|---|---|---|---|
| Top-1 accuracy (22 catalog items) | 86.4% | 90.9% | **95.5%** |
| Matched automatically | 17.9% | 56.4% | **51.3%** |
| Precision of automatic matches | 57.1% | 86.4% | **100%** (20/20) |
| Wrong automatic matches | 3 | 3 | **0** |
| Cost per item | ~$0 | $0.00014 | **$0.00019** |
| Mean latency per item | 264 ms | 823 ms | **1008 ms** |

**What the runs showed:**
- **Where the errors were.** The 3 remaining holdout errors were cases the regexes do not cover: a dimension written as `wys. 600 dł. 1000`, a missing pole count and a pump model. The LLM fixed two by picking the right candidate and rejected the third. It rejected none of the 12 correct high-similarity matches.
- **Threshold.** The confidence sweep is flat from 50 to 95, so the cutoff barely matters. The errors that remained were never in the LLM's own decisions.
- **Model choice.** `gpt-5-mini` was no better than `gpt-4.1-mini`, at about 2.5× the cost, several times the latency and with timeouts.
- **Limitations, stated plainly:**
  - **Small sample.** 39 holdout lines, so confidence intervals are wide.
  - **Not fully independent.** The holdout was written by the same author as the rules.
  - **Not fully deterministic.** The model is not deterministic even at temperature 0, and runs differ by 1–2 items. One known failure repeats: a `YDY 3x1.5` cable matched to the flat `YDYp` variant when the right cable was not among the candidates.
  - **Next step.** A run on a real customer estimate comes before enabling the layer in production.

---

## Margin model

- **Margin vs markup:** margin is `(sell − cost) / sell` and markup is `(sell − cost) / cost`. Each company picks its headline measure, and the other is always shown alongside. Entered percentages and alert thresholds follow the chosen basis.
- **Contract vs live price:** a quote's margin is calculated on the contract price the contractor committed to, while alerts fire on today's live price. Approving a quote freezes a snapshot, and the drift against live prices is what powers margin alerts.
- **Cost per estimate unit:** every cost figure is price divided by the unit factor, in margins, alerts, snapshots, baskets and the supplier optimizer.

---

## Multi-tenancy & security

- **Every query is tenant-scoped.** Every data-access function takes `tenantId` and every query filters on it. `tenantId` is read only from the verified JWT session, never from the request body.
- **A test enforces it.** It scans every SQL string passed to the database in `app/` and `lib/` (100+ queries) and fails on a data query without `tenant_id`, apart from a short reviewed list such as login, registration and migrations.
- **The Supabase public API is locked out of the database.** The `anon` and `authenticated` roles had their privileges revoked, and RLS is enabled on every table with no policies, which means default deny. The app connects directly as the database owner and is unaffected.
- **Brute-force protection lives in the database**, because serverless instances share no memory:
  - failed logins are limited per e-mail and per IP, and checked before bcrypt runs;
  - registrations are limited per IP;
  - the API answers 429 with `Retry-After`.
- **SQL is raw and parameterized** (`$1, $2, ...`) through `node-postgres`, with no string interpolation and no ORM.
- **Untrusted input is contained:**
  - uploaded files go through a file sandbox (type and size checks);
  - URL imports are size-capped while streaming and refuse private, loopback and link-local addresses (SSRF);
  - background jobs take the tenant from the job row, never from the payload.

---

## Data & background work

- **Database:** PostgreSQL 17 on Supabase with `pgvector` (semantic search) and `pg_trgm` (fuzzy lookup, dictionary trust).
- **Schema changes:** versioned, idempotent SQL migrations (15 so far), runnable as a script or an admin route.
- **Background work:** heavy ingestion (parsing large price lists, URL sync) runs through a background job processor and cron routes, not inside the request and response cycle.
- **Concurrency:** matching runs with bounded concurrency so large estimates finish within serverless time limits.

---

## Testing

- **Vitest suite (195 tests, pure logic, no database or network):**
  - LLM routing with a fake transport (confirmations, rejections, timeouts, untouched bands);
  - every correct pair and trap from the evaluation set, against Layer 2;
  - locale number parsing;
  - name normalization;
  - parameter extraction;
  - unit conversion;
  - margin and markup math;
  - dictionary trust and routing;
  - the tenant-filter guard.
- **Database-backed checks** (dictionary guard, units) run against the real schema inside transactions that are always rolled back.
- **Lint:** clean, with 0 errors and 0 warnings.

---

## Design principles (enforced throughout)

- TypeScript **strict**, no `any`.
- One responsibility per function, error handling on every async operation.
- No magic numbers. Constants live in `lib/config/` (thresholds, LLM, dictionary, auth).
- Parameterized SQL only.
- Tenant isolation is non-negotiable and checked by a test.

---

## Roadmap

- **LLM layer:**
  - repeat the evaluation several times to average out model variance;
  - measure on a real customer estimate before enabling the layer in production;
  - improve recall for cases where the right product is not among the top 5 candidates (`kabel` vs `przewód` wording).
- **Stripe billing.**
- **Supplier connectors:** broader URL connectors and scheduled sync.
