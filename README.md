# Geo-Insight: Humanitarian Funding Gap Monitor

A Databricks pipeline that ranks humanitarian plans by how far their funding falls short of documented need, with a dashboard and a natural-language (Genie) query layer on top.

*Built solo for the Databricks × UN OCHA Geo-Insight Challenge (CMU hackathon, May 2026), with AI coding assistance. **Selected as one of three winners.***

> *Which crises are most overlooked? And can we anticipate the next ones?*

---

## Headline finding

The **Lebanon slice of the Syria Regional Refugee & Resilience Plan (Syria 3RP, plan code LBN-RSYR25)** ranks as the most overlooked humanitarian plan of 2025 in this analysis, with the highest gap score and the largest unfunded amount of any plan:

- **$2.99B required · 9% funded · $2.72B unmet**

Four Tier 1 plans concentrate **$7.3B of overlooked need globally in 2025**:

| Plan code | Country | Plan family |
|---|---|---|
| LBN-RSYR25 | Lebanon | Syria 3RP (Regional Refugee & Resilience Plan) |
| COD-HCOD25 | Democratic Republic of Congo | Humanitarian Response Plan (HRP) |
| YEM-HYEM25 | Yemen | Humanitarian Response Plan (HRP) |
| MMR-HMMR25b | Myanmar | Humanitarian Response Plan (HRP) |

These are the most *overlooked* crises, not the most severe: Sudan has the highest severity in the data (9.6 of 10) but its plan is 40% funded, so it ranks just below, in Tier 2.

**What to watch next:** the four Tier 1 plans have one thing in common: **INFORM** (the Index for Risk Management's Severity Index, a standard monthly crisis-severity rating) rated their crises High or Very High in all or all but one of the past two years' monthly reports. Five countries show early signs of the same pattern:

- **Angola:** flagged worsening more often than any other country (Increasing in 10 monthly reports over the past year). Four more months at High would make it chronically severe by this project's definition.
- **Mexico:** the second most often flagged worsening (7 reports), though still rated Medium.
- **Libya** and **South Sudan:** already chronically severe, and still worsening.
- **Mauritania:** severity rising, but no humanitarian plan to fund.

These are watch signals, not forecasts; they have not been back-tested.

![Briefing dashboard — funding-gap KPIs and the tier-colored global map](dashboard-screenshots/briefing-page.png)

*The **Briefing** page: portfolio KPIs (combined shortfall, plans at risk, fastest-deteriorating country), a tier-colored global funding-gap map with marker size proportional to unmet need, and the natural-language **Ask Genie** surface — all responsive to the Year and Tier filters.*

---

## What this repo contains

```
humanitarian-funding-gap-monitor/
├── README.md                              ← you are here
├── WRITEUP.md                             ← technical write-up (design, architecture, evaluation)
├── LICENSE                                ← MIT
│
├── geo_insight_01_pipeline.ipynb          ← the complete Databricks pipeline (executed, all outputs)
│
├── gap_map.html                           ← interactive geospatial output
│
├── dashboard-screenshots/                 ← Databricks AI/BI Dashboard pages
│   ├── briefing-page.png
│   ├── deep-dive-p1.png
│   └── deep-dive-p2.png
│
└── genie-outputs/                         ← Databricks Genie-surfaced discovery findings
    ├── angola-trajectory.png               ← severity trajectory chart (shown below)
    ├── angola-sustained-drought.pdf
    ├── key-outlier-entities.pdf
    └── most-important-analytical-patterns.pdf
```

---

## The system

| Layer | Built in | What it does |
|---|---|---|
| **Pipeline** | Databricks notebook (this repo) | Six stages: Financial Tracking Service-as-backbone unified crisis table → INFORM Severity Index → gap score + tier classification → bonus task temporal signals → geospatial output → durable Parquet outputs |
| **Dashboard** | Databricks Artificial Intelligence / Business Intelligence (AI/BI) Dashboard | Three pages — **Briefing** (ranked table + map), **Deep Dive** (country KPIs + trajectory + compare), **Allocation** (leverage ranking + filter chain + override conditions) |
| **Genie Space** | Databricks Genie + Unity Catalog | Natural-language-to-SQL discovery surface over the same verified Parquet outputs |

![Deep Dive dashboard — Lebanon country drill-down](dashboard-screenshots/deep-dive-p1.png)

*The **Deep Dive** page (Lebanon, 2025): the headline finding made concrete — the Syria 3RP plan **RSYR25** at **9% funded · gap_score 0.61 · Tier 1**, shown beside a well-resourced flash appeal (78% funded, Tier 4). Same country, opposite ends of the neglect spectrum.*

---

## Pipeline outputs (Parquet tables)

The notebook produces the following Parquet outputs (durable in the Databricks workspace, available via Unity Catalog):

| Output table | Grain | Purpose |
|---|---|---|
| `ranked_gap_scores` | per-plan | gap_score, tier classification, plan-type-aware reasoning |
| `country_rollup` | per-country-year | severity, People in Need (PIN), funding %, aggregated gap_score |
| `documented_need_no_plan` | per-country | countries with documented severity but no formal humanitarian plan (e.g., Mauritania) |
| `inform_country_signals` | per-country | bonus task — 12- and 24-month signals from the monthly INFORM series (`structural_neglect_v2`, `months_increasing_12mo`) |

---

## Verification & quality

- **53 of 53 verification checks passing** (final notebook cell): a `check()` harness that pins the key outputs: table sizes, tier counts, the four Tier 1 plans, named country values (e.g., Sudan's 9.6 severity, Angola's 10 Increasing reports, the 1.98× rescale) and the saved output files.
- **Sensitivity:** with ±0.05 noise added to the inputs, at least 3 of the 4 Tier 1 plans stayed in Tier 1 in every trial (all 4 under severity-only noise). The wider top 10 is less stable (on average 7 to 8 of the 10 stay), so the project reports tiers, not exact ranks.
- **Coverage check:** INFORM rated 37 countries High or Very High in at least 18 monthly reports over the past two years; 36 of them have a plan in the ranking. The exception is North Korea, which has no plan in the Financial Tracking Service (FTS).

---

## What Genie surfaced

The Databricks Genie space was used to interrogate the verified Parquet outputs in natural language. Three discovery findings are included as PDFs in `genie-outputs/`.

![Angola INFORM severity, 2024–2026; the January 2026 jump reflects INFORM's change of scale](genie-outputs/angola-trajectory.png)

*Angola's INFORM severity, surfaced by Genie. The January 2026 jump is not a crisis: INFORM switched from a 1–5 to a 1–10 scale that month, and the pipeline detects this automatically (the global median doubles overnight). Angola's real warning sign is INFORM's own trend flag: Increasing in 10 of the past year's monthly reports, more than any other country. (Full finding: `genie-outputs/angola-sustained-drought.pdf`.)*

The three PDFs are kept as Genie produced them, unedited. Read them as raw model output; they contain these errors:

- **Angola** (`angola-sustained-drought.pdf`): "10 consecutive months" of worsening, February 2025 through April 2026. That window spans 15 months; the pipeline counts 10 Increasing reports, the figure used above.
- **A mislabeled plan** (both other PDFs): RRHOAY is called a "Rohingya response". It is the Regional Migrant Response Plan for the Horn of Africa to Yemen and Southern Africa.
- **A contradicted headline** (`most-important-analytical-patterns.pdf`): it reports an "inverse relationship" between severity and funding, then gives the correlation as weakly positive (0.17).
- **A wrong share** (same file): "10 countries account for 66%" of the funding gap, but its own figures ($15.6B of $29.36B) give 53%.

Genie is useful for exploring the data; its claims still need checking against the pipeline.

---

## Responsible AI stance

**No Large Language Model (LLM) in the scoring path.** The gap_score is a deterministic SQL/Python pipeline. LLM-based components (Databricks Genie) operate only on the **query surface** — translating natural-language questions into SQL against the verified Parquet outputs. Keeping the scoring deterministic is the project's main responsible-AI safeguard.

See `WRITEUP.md` §6 for details.

---

## Architecture in one sentence

The pipeline switches the join backbone from Humanitarian Response Plans (HRP) to the Financial Tracking Service (FTS), which expands scope from 22 countries to 77 and brings in regional plans like the Syria 3RP that an HRP-based join misses; it then computes a multiplicative `gap_score = severity_norm × need_norm × coverage_gap_norm` per plan, classifies plans into tiers, and checks coverage against a separate signal built from 4,927 monthly INFORM country ratings (108 countries).

---

## Datasets

All public, all via the Humanitarian Data Exchange (HDX):

| Dataset | Full name | Source |
|---|---|---|
| **FTS** | Financial Tracking Service | UN OCHA |
| **HNO** | Humanitarian Needs Overview | UN OCHA |
| **HRP** | Humanitarian Response Plans | UN OCHA |
| **INFORM Severity Index** | Index for Risk Management — Severity Index | ACAPS and the Joint Research Centre (JRC), European Union |

No proprietary data; no scraping; no API keys required.

---

## Glossary

| Term | Definition |
|---|---|
| **HRP** | Humanitarian Response Plan — country-level plan coordinated by UN OCHA |
| **HNO** | Humanitarian Needs Overview — UN OCHA's needs-assessment publication, source of `People in Need (PIN)` figures |
| **FTS** | Financial Tracking Service — UN OCHA's funding database; donor-self-reported with 3–9 month lag |
| **PIN** | People in Need — population identified as requiring humanitarian assistance |
| **Syria 3RP** | Syria Regional Refugee & Resilience Plan — covers 5 host countries: Lebanon, Jordan, Türkiye, Egypt, Iraq |
| **INFORM** | Index for Risk Management — humanitarian severity and risk scoring system (1–10 scale as of Jan 2026 rescale) |
| **op_env** | Operational Environment score — measures access difficulty (1 = open, 10 = maximum constraint) |
| **HDX** | Humanitarian Data Exchange — UN OCHA's open-data platform |
| **AI/BI** | Artificial Intelligence / Business Intelligence — Databricks' dashboard product family |
| **gap_score** | This project's composite mismatch metric: `severity_norm × need_norm × coverage_gap_norm` |
| **Tier 1** | gap_score ≥ 0.55 — the "most overlooked" set; exactly 4 plans in 2025 |

---

## Author

Author: [Asli Gulcur](https://asligulcur.github.io/).

Licensed under MIT (see `LICENSE`).
