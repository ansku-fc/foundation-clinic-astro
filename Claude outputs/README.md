# Foundation Clinic — Documentation hub (START HERE)

Single home for settled reference docs. If something here answers your question, it's
**decided** — don't re-derive it. Task-specific working briefs live in `docs/archive/`
(history, not reference). `CLAUDE.md` and the repo root `README.md` stay at the root.

## Map

### dashboard/
- **figures-and-formulas.md** — how every graph figure is computed (revenue/margin model,
  through-billing, KPIs, charts, source-of-truth map). The authority for calculations.
- **provisional-vs-actual.md** — SETTLED rules: which months use provisional vs actual, the
  banded ≤Aug 2026 / new-hourly ≥Sep 2026 comp boundary, doctor-comp single-source (no
  double-count) invariant, and the refresh + deploy chain. Read this before touching GM/GTV.
- **kpi-definitions.md** — human mirror of `src/data/kpi-definitions.ts` (edit the .ts first).
- **architecture.md** — app architecture, the two data layers, data pipeline.

### data-model/
- **database-schema.md** — Supabase schema.
- **query-cookbook.md** — common queries.
- **reconciliation.md** — GTV vs Netvisor reconciliation + revenue accruals.
- **purchase-structure.md** — purchase-invoice structure & validation rules.

### finance/
- **doctor-compensation.md** — comp model (banded + new hourly), bands, availability share.
- **invoicing.md** · **netvisor.md** · **netvisor-go-live.md** · **purchase-invoices.md** ·
  **netvisor-section-rules.md** — billing → Netvisor.

### scenarios/
- **assumptions.md** · **parameters.md** — projection model inputs.

### Run-the-scripts docs (live next to the code, linked here)
- Customers / memberships → `scripts/customers/README.md`
- Visits ingestion & reconciliation → `scripts/visits/README.md`
- Invoicing (statement → Netvisor draft) → `scripts/invoicing/README.md`
- Netvisor integration → `scripts/netvisor/README.md`
- Purchase-invoice reconciliation → `scripts/purchase-invoices/README.md`

## Data & deploy in one line
DB (Supabase) → export scripts → committed JSON in `src/data/` → Astro build → Vercel
(push to `main` auto-deploys). Vercel doesn't re-export; the nightly workflow
(`.github/workflows/refresh-dashboard-data.yml`, 03:00 UTC) refreshes the JSON and commits it.
Details: `dashboard/provisional-vs-actual.md` §5.

## Editing rule
To change a figure/KPI/margin definition, edit `src/data/kpi-definitions.ts` first, then keep
the matching doc, drawer, and formulas page in sync in the same change
(`fc-dashboard-figure-sync` skill).
