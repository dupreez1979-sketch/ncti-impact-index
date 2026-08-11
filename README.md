# Children's Theatre Impact Index

A live modelling and touring-subsidy system for the **National Children's Theatre Initiative**.

The Index scores every Australian community 0–10 for the social impact professional children's theatre could create there — using AEDC developmental data, ABS socio-economic disadvantage and remoteness — then turns that score into a transparent touring subsidy, builds tours against it, and produces funder-ready reports.

**Live:** [ctii.childrenstheatre.com.au](https://ctii.childrenstheatre.com.au)
**Access:** invitation only. There is no public or anonymous access.

---

## Contents

- [What it does](#what-it-does)
- [How the Index works](#how-the-index-works)
- [Repository contents](#repository-contents)
- [Architecture](#architecture)
- [First-time setup](#first-time-setup)
- [Deploying](#deploying)
- [Making changes to the app](#making-changes-to-the-app)
- [Database schema](#database-schema)
- [Access control](#access-control)
- [Reports](#reports)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Data sources](#data-sources)
- [Further reading](#further-reading)

---

## What it does

**Index** — score, filter and map 161 venues and 2,353 statistical areas nationally. Adjust the model live (indicator weights, geography, tier thresholds, subsidy ramp) and watch scores, tiers, subsidy rates, maps and charts recompute instantly. List, Map and Analysis views share one filter set.

**Tours** — a portfolio database of every tour across every company: venues reached, subsidy committed, children reached, with List, Map and Analysis views. Each tour freezes the model it was built under, so agreed subsidy figures never shift when the national model is refined.

**Tour building** — activate a tour and the main venue list becomes the editor: tick venues on or off the tour, set per-venue fees, confirm subsidy. Changes stage locally and commit on **Save**.

**Reports** — four print-ready PDFs: model specification, tour proposal, funder subsidy report, and a venue-facing offer letter.

**Users** — email/password sign-in with two roles (Admin, Viewer), enforced in the database.

---

## How the Index works

Each indicator is transformed to a 0–10 sub-score where **5 = the national average**, then combined as a weighted mean:

| Indicator | Sub-score | Default weight |
|---|---|---|
| Emotional maturity (AEDC 2024) | `min(10, EM% ÷ 25.17 × 5)` | 40% |
| Disadvantage (ABS SEIFA 2021, IRSD) | `11 − decile` | 40% |
| Remoteness (ABS ASGS 2021) | Major Cities 0 · Inner 2.5 · Outer 5 · Remote 7.5 · Very Remote 10 | 20% |
| Vulnerable on 2+ AEDC domains (DV2) | `min(10, DV2% ÷ 12.46 × 5)` | off by default |

```
Index = Σ(weightᵢ × sub-scoreᵢ) ÷ Σ(weightᵢ)     … indicators with no data are skipped
```

**Tiers:** High ≥ 6.3 · Medium ≥ 3.8 · Low below.
**Subsidy** (High tier only): ramps 50% → 100% between the High threshold and index 8.0, rounded to 5%.

Every threshold above is a model parameter, not a constant. See the [technical paper](#further-reading) for the reasoning, limitations and known biases.

---

## Repository contents

```
index.html            The entire application — single file, no build step
classic.html          Pre-redesign layout, kept for reference
netlify.toml          Netlify config (publish root, no build command)
supabase/
  schema.sql          Core tables, venue_full view, base RLS
  auth.sql            Profiles, roles, locked-down RLS, storage policy
  models.sql          Saved models
  tours.sql           Companies → shows → tours → tour_venues
  dv2.sql             DV2 columns + data + updated view
  fix-admin.sql       Utility: (re)grant admin to a known email
  AUTH-SETUP.md       Step-by-step auth configuration
```

**Not in this repo:** `gen_dashboard.py` (the generator that produces `index.html`) and the source CSVs live in the private working folder — the CSVs contain the full dataset and are excluded by `.gitignore`. This repo holds the deployable artefact.

---

## Architecture

```
Browser ── index.html (single file, ~390 KB)
              │  Leaflet + Chart.js from CDN
              │  supabase-js for auth and writes
              ▼
         Supabase (PostgreSQL)
              ├─ PostgREST  → venue_full, sa2_scores, census, models, tours
              ├─ Auth       → email/password, roles in `profiles`
              └─ Storage    → sa2_boundaries.geojson (~95 MB, private bucket)
```

**No build step, no framework, no bundler.** State lives in memory; scoring all 2,353 areas takes under a millisecond, which is what makes live parameter dragging feel instant.

**Sub-scores are pre-computed** at ingestion and stored in `sa2_scores` for both geographies. The browser only applies weights and thresholds. Re-run the data pipeline when a source dataset is revised.

**Boundary geometry loads asynchronously** after the dashboard is interactive, so the 95 MB GeoJSON never blocks analysis. A skeleton screen with a progress bar covers the initial load.

---

## First-time setup

Run these once against a fresh Supabase project, in order.

### 1. Schema and data

```
SQL Editor → run supabase/schema.sql
```

Then import the CSVs in this order (Table editor → Import data from CSV), because venues reference SA2s:

| # | File | Table | Rows |
|---|---|---|---|
| 1 | `sa2_scores.csv` | `sa2_scores` | 2,353 |
| 2 | `census_sa2.csv` | `census_sa2` | 2,472 |
| 3 | `census_sa3.csv` | `census_sa3` | 358 |
| 4 | `sa2_geometry.csv` | `sa2_geometry` | 2,454 |
| 5 | `app_config.csv` | `app_config` | 2 |
| 6 | `venues.csv` | `venues` | 161 |

Then run `models.sql`, `tours.sql` and `dv2.sql`.

### 2. Boundary geometry

Storage → create bucket **`geo`** → upload `sa2_boundaries.geojson` → set the bucket **private**.

### 3. Authentication

Follow **`supabase/AUTH-SETUP.md`**. In summary:

- Authentication → Email: turn **Confirm email off**, leave **Allow sign-ups on** (the Users screen needs it; self-signed-up strangers get no access because they have no `profiles` row).
- Authentication → URL Configuration: add the site URL to the redirect allow-list, or password-reset emails will fail.
- Run **`auth.sql`** — creates roles, locks every table to approved users, restricts writes to admins, and seeds the first admin.
- Give yourself a password: Authentication → Users → your user → **Send password recovery**.

### 4. Keys

The app embeds the project URL and the **publishable** key. That is safe: every table is governed by row-level security, so the key alone grants nothing without an approved login. **Never** put the `service_role` key in the client.

---

## Deploying

Netlify builds from this repo — publish directory `.`, no build command.

```bash
git add index.html
git commit -m "Update dashboard"
git push
```

Every push to `main` redeploys. To roll back, revert the commit — `index.html` is self-contained, so any past commit is a working build.

---

## Making changes to the app

`index.html` is **generated**. Do not hand-edit it — edit `gen_dashboard.py` in the working folder and rebuild:

```bash
python3 gen_dashboard.py          # writes Childrens_Theatre_Impact_Index_Dashboard.html
node --check /tmp/app.js          # syntax check of the extracted JS
cp Childrens_Theatre_Impact_Index_Dashboard.html ncti-impact-index/index.html
```

The generator embeds the fonts and both logos as base64 and substitutes `__SBURL__` / `__SBKEY__`, so the output has no external assets beyond the CDN libraries.

---

## Database schema

| Table | Purpose |
|---|---|
| `venues` | The editable venue master — name, state, status, postcode, coordinates, SA2/SA3 assignment |
| `sa2_scores` | Pre-computed sub-scores per SA2 at both geographies (`e2/d2/r2/v2`, `e3/d3/r3/v3`), child counts, has-venue flag |
| `census_sa2` / `census_sa3` | ABS Census 2021 contextual profile (12 fields) — displayed, never scored |
| `sa2_geometry` | SA2 boundary polygons |
| `app_config` | Australia locator outline, default model parameters |
| `models` | Saved models: name, description, config JSON |
| `companies` → `shows` → `tours` → `tour_venues` | The tour hierarchy; `tours.model` holds the frozen snapshot, `tour_venues` the per-venue fee and confirmation |
| `profiles` | Who may sign in, and their role |
| `venue_full` (view) | The joined, nested shape the app reads in one query |

---

## Access control

Two roles, enforced by row-level security — not merely hidden in the interface:

| | Viewer | Admin |
|---|---|---|
| See index, maps, tours, reports | ✅ | ✅ |
| Edit venues, models, tours | ❌ | ✅ |
| Manage users | ❌ | ✅ |

A viewer's write is rejected by the database regardless of what the client does. Admins add users (☰ → Users) with a temporary password and an optional forced password change on first login; revoking a user removes data access immediately.

---

## Reports

All generated client-side and print-ready (Print → Save as PDF).

| Report | For | Contains |
|---|---|---|
| **Model report** | Analysts | Weights, formulas, thresholds, subsidy ramp, tier distribution, subsidy ladder, index distribution, national counts |
| **Tour report** | Funders | Subsidy sought, presenter contribution, tour value, funding sources donut, venues by state, national map |
| **Funder report** | Funders | The same for an ad-hoc venue selection |
| **Venue report** | Presenters | Their score with indicator bars, position on the subsidy ramp, the offer, children reached |

Every report states the model it was produced under.

---

## Testing

The app is covered by a headless-DOM suite (jsdom) — roughly 120 assertions across model mathematics, saved-model backwards compatibility, tour staging and save semantics, role-based access, report generation, responsive layout and interface state machines.

```bash
npm install jsdom
node tourmode.js && node authtest.js && node winstate.js && node dv2test.js && node reports.js && node skel.js
```

Run these against a fresh build before pushing.

---

## Troubleshooting

**Login fails for a known user** — the account may have no password (older magic-link accounts). Authentication → Users → Send password recovery.

**"Awaiting approval" after signing in** — the account has no `profiles` row. Add them via ☰ → Users, or run `fix-admin.sql` for an admin.

**Reset emails never arrive** — the site URL is missing from Authentication → URL Configuration. Also note the 60-second rate limit.

**Map shows pins but no shading** — boundary geometry failed. Check the `geo` bucket exists, contains `sa2_boundaries.geojson`, and that `auth.sql`'s storage policy was applied.

**Empty tables after login** — RLS is blocking reads; confirm `auth.sql` ran and the user has a `profiles` row.

**Save fails on a tour** — should not occur; if it does, reload (staged changes are discarded, nothing is written) and report it.

---

## Data sources

| Source | Vintage | Use |
|---|---|---|
| Australian Early Development Census | 2024 | Emotional maturity, DV2 |
| ABS SEIFA (IRSD) | 2021 | Disadvantage decile |
| ABS ASGS — Remoteness Areas | 2021 | Remoteness class |
| ABS ASGS — SA2/SA3 boundaries | 2021 | Geography, mapping |
| ABS Estimated Resident Population | latest | Children 0–14 by age band |
| ABS Census, General Community Profile | 2021 | Contextual venue profile only |

---

## Further reading

- **`Children's Theatre Impact Index — Technical Paper.md`** (working folder) — full methodology, transformations, limitations, validation
- **`supabase/AUTH-SETUP.md`** — authentication configuration
- **`UX Redesign Plan.md`** (working folder) — the interface architecture and its rationale

---

© 2026 National Children's Theatre Initiative — *Theatre for Every Child*
