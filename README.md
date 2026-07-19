# Margio

**B2B SaaS that matches construction cost-estimate items to wholesaler price lists and protects the contractor's margin in real time.**

Electricians, plumbers and construction firms bid on jobs using cost estimates full of messy material names. Getting a real price means matching each line against several wholesaler price lists, by hand, in Excel, for hours, with mistakes that quietly eat the margin. Margio automates that matching and flags every line where the margin drops below target.

![Margio dashboard](docs/screenshots/dashboard.png)

> 🛠 **Status:** engineering thesis project and a commercial product in active development.
> 🚧 **Actively built and tested.** The app is continuously improved, so some features are still evolving and parts of the live demo may be rough around the edges.
> 🔒 **Closed-source.** The code is private. This repository is a technical case study.
> ▶️ **Live demo:** [margiotest.vercel.app](https://margiotest.vercel.app) &nbsp;·&nbsp; demo login: `demo@margio.app` / `demodemo`
> 💬 Happy to walk through the code and architecture in an interview.

---

## What it does

- **Ingest price lists** from file upload (CSV / XLSX) or a supplier URL, with **automatic column mapping**. The app figures out which column is SKU, EAN, name, price or unit, even when every supplier formats differently.
- **Match estimate items to catalog products** through a layered pipeline (exact, then parameter extraction, then semantic search) with a **confidence score** on every match.
- **Human-in-the-loop** for uncertain matches. The user's decision is saved to a per-company dictionary, so the system gets smarter for that tenant over time.
- **Real-time margin alerts.** When a supplier price change pushes a line below the target margin, the affected projects are flagged.
- **Supplier optimization.** Split a basket across suppliers to minimize total cost.
- **Export** supplier-split quotes to **Excel and PDF** (in Polish, for the end client).
- **Project snapshots and diffs.** Freeze a quote and see exactly what changed when prices move.
- **Multi-tenant.** Every company's data is fully isolated.

---

## Architecture at a glance

Margio is a **Next.js (App Router) + TypeScript** application backed by **PostgreSQL (Supabase, PG 17 + pgvector)**, deployed on **Vercel**. There is no ORM. Data access is **raw parameterized SQL** through `node-postgres`, kept in typed query functions. Authentication is a **custom JWT + bcrypt** implementation.

The interesting part is the **matching engine**, a deterministic-first, AI-last pipeline. See **[ARCHITECTURE.md](ARCHITECTURE.md)** for the full breakdown. The short version:

```
Estimate item
   │
   ▼
Layer 0  Normalization       parse CSV/XLSX, clean EAN/SKU, unify units
   ▼
Layer 1  Exact Match         EAN → SKU → company dictionary → normalized name
   ▼
Layer 2  Parameter Extract   pull technical params (cross-sections, diameters), no AI
   ▼
Layer 3  Semantic Search     pgvector similarity over catalog embeddings
   ▼
Confidence Aggregator        scores the candidates and routes the item:
   ├─ high     → AUTO_ACCEPT
   ├─ medium   → LLM re-ranking (planned) or human review
   └─ low      → Human-in-the-loop → company dictionary
```

**Why this order matters:** the cheap, deterministic layers resolve the easy majority of items first, so the expensive, fuzzy layers only run on what is genuinely ambiguous. It is faster, cheaper and far more predictable than throwing every line at a vector search or an LLM.

---

## Engineering highlights

**Hybrid matching, not "just embeddings."**
A pure semantic search confuses a 3×2.5mm² cable with a 3×1.5mm² one. They look similar, they are technically different, and a wrong match corrupts a real quote. So exact identifiers (EAN/SKU) and extracted technical parameters run before vector search, and semantic similarity only breaks ties within the same parameter class.

**Confidence-based routing with a human fallback.**
Every item gets a confidence score that decides its path: auto-accept, re-rank or human review. Uncertain items never fail silently. They surface to the user, and each manual decision is written back to a per-company dictionary so the same item resolves automatically next time. Matching also skips archived and superseded price lists and picks deterministically (newest valid entry first), so re-running a match gives stable, explainable results rather than shifting under the user.

**Margin integrity: contract price vs live price.**
A quote's margin is calculated on the **contract price** the contractor committed to, while alerts fire on **today's live price**. These are deliberately two separate cost bases. Approving a quote locks in the margin the user agreed to, and a separate live-cost check still warns "if you bought this today, you'd be below your threshold." Collapsing them into one number would either freeze the margin and silence drift alerts, or let an agreed quote's numbers move under the contractor's feet. Keeping them split is what makes both the committed margin and the drift alert trustworthy.

**Strict multi-tenant isolation.**
Every data-access function takes `tenantId`, and it is **only ever read from the verified JWT session, never from the request body.** No query that touches company data runs without a tenant filter. This is enforced as a hard rule across every route that reads or writes tenant data.

**Prices stay out of the AI.**
By design, the planned LLM re-ranking layer receives product candidates but **never sees prices.** Pricing logic stays deterministic and auditable, and the model cannot be nudged by cost.

**Safe ingestion of untrusted files.**
Uploaded price lists are validated and processed inside a file sandbox (type checks, size limits), and heavy ingestion runs through a background job processor rather than blocking the request.

**Price-list dedup, snapshots and diffs.**
Migrations add deduplication, trigram indexing for fast fuzzy lookup, and project snapshots, so a quote can be frozen and diffed against live prices later.

---

## Tech stack

| Layer | Choice |
|---|---|
| Framework | Next.js 16 (App Router), React 19, TypeScript (strict) |
| Database | PostgreSQL 17 on Supabase, `pgvector`, `pg_trgm` |
| Data access | `node-postgres` (raw parameterized SQL, no ORM) |
| Auth | Custom JWT (`jose` / `jsonwebtoken`) + bcrypt, multi-tenant |
| AI | OpenAI embeddings (semantic search); optional LLM re-ranking on the roadmap |
| Files | `papaparse`, `xlsx`, `exceljs`, `@react-pdf/renderer` |
| Deploy | Vercel |

---

## About

Built by **Bartłomiej Dobranowski**, Computer Science student (Erasmus+ in Porto).
Margio is my engineering thesis and a product I am developing toward launch.

- GitHub: [@xtfee](https://github.com/xtfee)
- LinkedIn: [bartlomiej-dobranowski](https://www.linkedin.com/in/bartlomiej-dobranowski/)
- Email: ecomdobranowski@gmail.com

*The source code is private. This repository documents the architecture and decisions. I am glad to walk through the actual codebase live.*
