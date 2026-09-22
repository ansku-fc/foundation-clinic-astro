# Provisional vs. actual — when the dashboard uses which (SETTLED)

This is the settled rule for which data source drives each month on the GTV and
gross-margin graphs. If you're about to re-derive "why does month X show provisional/
actual" or "which doctor-comp model applies" — read this first, it's already decided.

Canonical formula detail: `docs/dashboard/figures-and-formulas.md` (§1, §5, §6).
This page is the decision rules and invariants layered on top.

---

## 1. Revenue / GTV — always actual-invoiced, one exception

Revenue and GTV are recognised on an **actually-invoiced, invoice-date basis** for every
month, read from `invoice_lines`. There is no "projection" for a closed month's revenue.

The single exception is the **open cycle** (the not-yet-invoiced current month), which uses
`get_provisional_cycle` → `provisional.json`. The open cycle is defined by data, not code:

```
open cycle = margin_rules.sales_closed_through + 1
```

So GTV auto-tracks `sales_closed_through` — no hardcoded month. Nothing to maintain here.

## 2. Cost / gross margin — three month-states

Cost (and therefore GM%) is the part with states. A month is exactly one of:

| State | When | Doctor cost source | Other cost | Rendered |
|---|---|---|---|---|
| **Final** (accrual) | month ≤ `margin_pipeline_end_sm` (inside `margin_export.blendedCm.window`) | `doctor_payments` actuals, via blended CM | audited blended accrued (Supabase margin views) | solid |
| **Preliminary** | past the window, not yet the open cycle | `doctor_payments` actual **if rows exist**, else the live comp engine | nurse from `nurse_cogs_config`; lab/pharma from cost-month purchase invoices (lag → self-corrects) | faded |
| **Provisional** (open cycle) | month == `sales_closed_through + 1` | live comp engine (`get_doctor_comp_month`) | DB-actual where landed + trailing-3-month run-rate for what hasn't | faded/dashed |

Transitions (all data-driven, no code edit intended):
- **Provisional → Preliminary** when `sales_closed_through` advances past the month.
- **Preliminary → Final** when the month enters the blended-CM window (`margin_pipeline_end_sm`).

A month flips from estimate to actual **the moment its `doctor_payments` rows exist** — that's
why recording the doctor invoices (below) is what makes a month's GM real.

## 3. Doctor-comp model boundary (SETTLED)

```
service month ≤ 2026-08  → BANDED model
service month ≥ 2026-09  → NEW HOURLY model
```

- Banded months: actual from `doctor_payments`; for a banded month not yet invoiced, the live
  banded figure is `get_doctor_comp_draft(doctor, month)`.
- New-model months: `get_doctor_comp_month(month)`.
- **Do not apply the new hourly model to a month ≤ 2026-08.** Doing that to Aug 2026 overstated
  GM by ≈11pp (used €7,806.19 instead of the €12,563.94 actually invoiced).
- Encode the boundary as config (planned: `margin_rules.doctor_comp_model_switch_sm = '2026-09'`),
  not as hardcoded per-month sets.

## 4. Invariants — do not violate (each caused real rework)

1. **Doctor comp has ONE source of truth per month.** For any month whose comp is recorded in
   `doctor_payments`, that is the figure — and the Netvisor / Health-P&L path must **exclude**
   doctor comp for that month. Never let both contribute, or the month double-counts doctor cost.
   (This is why we did NOT extend `health_pl_end_sm` into a month whose doctor invoices aren't
   in Netvisor.)
2. **`doctor_payments.model = 'hourly'` is the label for the BANDED model** on 2026-02+ records
   (historical misnomer, documented in `comp_model.json`). Five consumers branch on `'hourly'`
   (margins.tsx, onboarding-cohorts.ts, doctor-hours.ts, …). Do **not** introduce a `'banded'`
   value without updating all of them.
3. **Revenue stays invoice-date actual** for every month. Never switch revenue to a projected basis.
4. The margin assembly is **overwrite, not additive** (blended loop then PRELIM loop, last writer
   wins) — a month can't be summed from two paths. Keep it that way.
5. To change any figure definition: edit `src/data/kpi-definitions.ts` first, keep drawers +
   `docs/dashboard/kpi-definitions.md` + this file in sync in the **same** change
   (see the `fc-dashboard-figure-sync` skill).

## 5. How figures reach the live dashboard (deploy chain)

```
Supabase (DB)  →  export scripts  →  committed JSON in src/data/  →  Astro build  →  Vercel (push to main auto-deploys)
```

Vercel builds have **no `.env`**, so they do NOT re-export — they build from committed JSON.
**A DB change is invisible on the live site until the JSON is re-exported AND committed.**

Two ways the JSON refreshes:
- **Nightly, automatic:** `.github/workflows/refresh-dashboard-data.yml`, cron `0 3 * * *`
  (03:00 UTC ≈ 06:00 Helsinki). Runs `generate-data`, `export-margins`, `export-provisional`,
  `export-doctor-payments`, `export-group`; commits `src/data` if changed; Vercel redeploys.
  Also runnable on demand (Actions → Run workflow).
- **Manual, local** (needs `.env` with `SUPABASE_URL` + `SUPABASE_SERVICE_ROLE_KEY`):
  `npm run generate-data && npm run export-margins && npm run export-provisional &&
  npm run export-doctor-payments && npm run export-group`, then commit + push.

Debugging "I changed the DB but the dashboard didn't move": the JSON hasn't been re-exported/
committed yet — wait for the nightly, or run the exports and commit.

## 6. Verified reference numbers — Aug 2026 (banded actual)

So these don't get re-derived:

| doctor | invoice | total |
|---|---|---|
| enni (Dolor Oy / Enni Sanmark) | 307 | €4 696.62 |
| anni (Anni Karjala tmi) | 407 | €1 900.00 |
| eevert (Eevert Partinen tmi) | 507 | €3 688.60 |
| jesper (Mediorem Oy / Jesper Rautiola) | 607 | €2 278.72 |
| meeri | — (0 clinical hours) | €0 |
| **Σ doctor comp** | | **€12 563.94** |

August GM ≈ **39.5%** (net revenue ≈ €41 656.24; total cost ≈ €25 219.10). `doctor_payments`
rows: `service_month = 2026-08`, `model = hourly` (=banded), `source = invoice`,
`payment_month = 2026-09`, `payment_id = dp-<doctor>-202609`.

> Status: the data-driven automation of §2 (replacing the hardcoded `PRELIM_MONTHS`/`_PRELIM_COST`
> with the presence/boundary rule) is the "Part 2" refactor — planned, not yet implemented as of
> 2026-09-14. Until it lands, the rule above describes intended behavior; the current code still
> carries `PRELIM_MONTHS = {2026-08}`.
