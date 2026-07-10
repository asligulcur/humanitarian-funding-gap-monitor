# Geo-Insight: Technical Writeup

**A submission to the UN OCHA Geo-Insight Challenge — May 2026.**

Author: Asli Gulcur · Powered by Databricks

---

## 1. Question and headline finding

**The challenge:** which humanitarian crises are most overlooked — defined as the mismatch between documented need and humanitarian financing?

**Headline finding:** the **Lebanon slice of the Syria 3RP (LBN-RSYR25)** is the world's most-overlooked humanitarian plan in 2025. At $2.99B required, 9% funded, and $2.72B unmet, it ranks #1 by gap_score (0.61). Four Tier 1 plans concentrate **$7.3B of overlooked need globally**: Lebanon (Syria 3RP), DRC HCOD25, Yemen HYEM25, Myanmar HMMR25b.

None of these is the world's worst crisis by severity — Sudan is — but each is severely underfunded relative to documented need. *Worst crisis is not the same as most overlooked. That distinction is itself the central methodological finding.*

**The anticipatory dimension:** today's most-overlooked plans followed documented multi-year trajectories. The same trajectory signals are visible now for five named rising-risk countries: Angola, Mauritania, Mexico, South Sudan, Libya. Angola is four months from the structural-neglect threshold. Acting on those signals before they crystallize in the single-snapshot gap_score IS anticipatory action in OCHA's formal sense.

---

## 2. System architecture

### 2.1 FTS-as-backbone — the architectural choice

The conventional humanitarian funding-gap analysis joins HRP (Humanitarian Response Plans) with HNO (Humanitarian Needs Overview) data. This covers ~22 countries with formal HRPs.

We made the architectural choice to use **FTS (Financial Tracking System) as the join backbone instead**, which captures:

- **Flash Appeals** (e.g., Palestine FPSE25, Lebanon LBP25)
- **Regional Response Plans** (e.g., the Syria 3RP host-country slices, the Ethiopia regional plan)
- **HRPs** (the conventional set)

**Result:** scope expands from 22 to 77 countries. The Syria 3RP, the world's most-overlooked plan, becomes visible — it was invisible to any HRP↔HNO join.

### 2.2 Three output tables

The pipeline produces three Parquet outputs, each focused on a single analytical question:

| Table | Grain | Purpose |
|---|---|---|
| `ranked_gap_scores` | per-plan | gap_score, tier classification, plan-type-specific reasoning |
| `country_rollup` | per-country-year | severity, PIN, funding %, aggregated gap_score |
| `documented_need_no_plan` | per-country | countries with documented severity but no formal plan (e.g., Mauritania) |

**Discipline:** no single table is forced to carry a decision the data can't support. The dashboard surfaces the right table for each persona's question.

### 2.3 Bonus task — temporal layer

A separate analytical pathway (Stage 4 of the notebook) computes monthly trailing-window signals on the INFORM Severity Index:

- **4,927 monthly observations** across **108 countries** (Jan 2021 → April 2026)
- `structural_neglect_v2` flag: country at INFORM High+ for ≥18 of last 24 months → **37 countries flagged**
- `inform_severity_trend`: 3-month directional classification (Increasing / Stable / Decreasing)
- Scale-invariant via categorical labels (handles the Jan 2026 INFORM rescale from 1–5 → 1–10)

**Cross-validation result:** 36 of 37 chronic-neglect countries already appear in the gap ranking. Only DPRK is invisible (no formal humanitarian plan tracked in FTS). This is independent confirmation that the FTS-as-backbone architecture isn't missing systematically neglected countries.

The temporal layer surfaces rising-risk signals **on the order of 6–12 months before** the single-snapshot gap_score would flag them — a heuristic estimate based on the trailing-window structure (12-month and 24-month lookbacks), not a directly backtested measurement.

### 2.4 Dashboard design — the user-facing layer

The Databricks Artificial Intelligence / Business Intelligence (AI/BI) Dashboard is the user-facing layer of the system. Three pages, each calibrated to a specific humanitarian persona's decision-making workflow:

| Page | Persona | Decision the page supports |
|---|---|---|
| **Briefing** | Humanitarian Coordinator + donor advisor | Which crises require brief-level attention; ranked table + global map + 10 deterministic Quick Questions + integrated Ask Genie button |
| **Deep Dive** | Humanitarian Affairs Officer defending a brief | Evidence trail for a specific country (KPIs, provenance trace, multi-year coverage trajectory, compare-to feature) + the "What-If" calculator |
| **Allocation** | Country-Based Pooled Funds (CBPF) Manager | Leverage ranking (severity × marginal coverage gain per dollar) with five named override conditions |

**The "What-If" calculator** on the Deep Dive page is intentionally **mechanical arithmetic, not a forecast**. Typing an incremental funding amount shifts the funding % and recalculates the tier deterministically via the same gap_score formula. It does not model donor behavior or absorption capacity.

**The Allocation page's five override conditions** — access negotiations, severity context, sector-level plays, donor earmarking, structural neglect — let the decision-maker deviate from the leverage rule with full transparency. The system surfaces the analytical default; the user surfaces the human judgment.

Sample screenshots: `dashboard-screenshots/briefing-page.png`, `deep-dive-p1.png`, `deep-dive-p2.png`.

### 2.5 Genie Space — discovery surface

Databricks Genie operates as a **natural-language-to-SQL discovery surface** over the verified Parquet outputs. Analysts ask questions in plain English; Genie generates SQL against Unity Catalog-governed tables; results return as charts or tables.

Domain instructions guide the LLM on:

- Humanitarian terminology (plan codes, severity categories, operational environment scoring)
- Year-handling for multi-year comparisons (closed years vs partial-year year-to-date)
- The Jan 2026 INFORM Severity Index scale change (1–5 → 1–10) — categorical comparisons preserved across the rescale

**What this enables:** discovery of patterns not pre-computed by the dashboard. The four systemic-forces analysis cited throughout this writeup (donor fatigue, plan-type bias, visibility bias, requirements-mask-cuts) was assembled from Genie queries against the verified outputs. Sample Genie discovery outputs are included in this repo: `genie-outputs/`.

**Architecturally critical:** the LLM operates ONLY on the **query surface**. It generates SQL; it never modifies the gap_score logic, tier thresholds, or output tables. Every Genie result is reproducible by running the generated SQL directly. This is the operational core of the responsible-AI stance described in §6.

---

## 3. Gap score design

### 3.1 The formula

```
gap_score = severity_norm × need_norm × coverage_gap_norm
```

Where:

| Component | Definition |
|---|---|
| `severity_norm` | INFORM Severity Index / 10 (range 0–1) |
| `need_norm` | Log-scaled need. HRPs use HNO PIN; Regional/Flash plans use FTS requirements as proxy (plan-type-aware) |
| `coverage_gap_norm` | `1 − fts_pct_funded / 100`. Annualized variant applies for in-progress years to avoid penalizing plans that simply haven't received late-year donor commitments yet. |

Tier thresholds:

| Tier | Range | Meaning |
|---|---|---|
| Tier 1 (most overlooked) | `gap_score ≥ 0.55` | Severely underfunded relative to need; sustained |
| Tier 2 | `0.45 ≤ gap_score < 0.55` | Threshold edge — sensitive to noise |
| Tier 3 | `0.30 ≤ gap_score < 0.45` | Underfunded but less acute |
| Tier 4 | `gap_score < 0.30` | Adequately resourced relative to need |

### 3.2 Why multiplicative, not additive — the deliberate design choice

A multiplicative formula encodes **mismatch**: all three signals (severity, need, coverage gap) must be high *simultaneously* for a plan to score high. If any component is near zero, the score is near zero.

An additive formula would collapse to **severity-weighted ranking with funding as a modifier** — different question.

**The Sudan paradox is the test case:**

- Sudan has the world's highest INFORM severity (**9.6**) and largest PIN (**30.4M**).
- Yet Sudan ranks **Tier 2, not Tier 1**.
- Why: at 40% funded, Sudan is better-resourced relative to need than the four Tier 1 plans (all below 30% funded).

Under a *hypothetical* additive formulation, severity would dominate the sum and Sudan would likely rank #1 (its INFORM severity of 9.6 is the highest globally). Under our multiplicative formulation, Lebanon RSYR25 ranks #1 (mismatch dominates: high severity AND high need AND high coverage gap simultaneously).

*Note: we did not actually compute the additive version on our data. This is a structural argument about the formula's behavior, not an empirical comparison.*

Both are valid analytical choices answering **different questions**. We chose multiplicative because the challenge asks for *mismatch* (overlooked relative to need), not severity. The phrase "worst crisis ≠ most overlooked" is the central methodological insight that the multiplicative formula produces.

### 3.3 Plan-type-aware need normalization

HRPs and Regional/Flash plans report needs differently. HRPs publish HNO PIN figures; Regional and Flash plans use FTS requirements (in USD) as the only consistent need proxy. The pipeline normalizes each plan-type independently to prevent need-scale artifacts from dominating the gap_score.

Effect on results: the plan-type bias finding (regional plans avg 21% funded vs HRPs at 34%) emerges from this distinction. The `gap_score` doesn't double-penalize regional plans for being regional; it surfaces the structural underfunding cleanly.

---

## 4. Evaluation methodology

### 4.1 Verification: 53 of 53 tests passing

The `verification_test` cells in the notebook run 53 deterministic checks against the Parquet outputs:

- **Schema integrity** — all expected columns present, correct types, no nulls in primary keys
- **Row counts** — no plans dropped silently between stages
- **Tier consistency** — Tier 1 = exactly 4 plans in 2025 (named: LBN-RSYR25, COD-HCOD25, YEM-HYEM25, MMR-HMMR25b)
- **Named-finding consistency** — Lebanon RSYR25 gap_score = 0.61, ranks #1; Sudan = Tier 2 with 40% funded; HYEM26 at 0.55 (threshold edge)
- **Cross-table consistency** — `country_rollup` aggregations match per-plan sums from `ranked_gap_scores`
- **Time series integrity** — INFORM monthly data has no unexplained gaps; coverage extends Jan 2021 → April 2026

**All 53 checks pass.** Every claim in the slide deck and this document traces to a verifiable row in `gap_score_v1_ranked.parquet`.

### 4.2 Sensitivity analysis

We perturbed each gap_score component with ±0.05 noise and measured top-10 plan preservation over 5 trials:

| Perturbation | Top-10 preserved (avg of 5) | Above 8.0 robust threshold? |
|---|---|---|
| severity-only | 8.4 / 10 | ✅ yes |
| need-only | 7.8 / 10 | ❌ no |
| coverage-only | 7.8 / 10 | ❌ no |
| all-three combined | 6.6 / 10 | ❌ no |

**Within-tier ordinal rank churns under noise. The Tier 1 SET (which four plans qualify) holds 3–4 of 4 plans throughout.**

We therefore defend the **tier set**, not the within-tier order. The slide deck's Tier 1 bar chart shows the four Tier 1 plans + HYEM26 (Tier 2 threshold edge) in orange to signal robustness boundaries honestly.

### 4.3 Bonus task as independent validation

The temporal layer (§2.3) was constructed as an independent analytical pathway — different inputs (multi-year INFORM time series instead of single-year FTS+HNO+INFORM cross-section), different aggregation (trailing-window signals instead of point-in-time gap_score).

Cross-checking: 36 of 37 chronic-neglect countries already appear in the gap ranking. The single miss (DPRK) is structurally invisible (no formal humanitarian plan in FTS). This is independent confirmation that:

1. The FTS-as-backbone architecture isn't missing chronically-neglected countries
2. The gap_score correctly surfaces sustained underfunding (not just acute spikes)
3. The temporal layer adds forward visibility (rising-risk signals) without contradicting the cross-sectional ranking

---

## 5. Limitations and systematic biases

### 5.1 Seven systematic biases we surface honestly

| Bias | What it produces |
|---|---|
| **Donor attention** | Geopolitically visible crises (Ukraine, Sudan) attract attention disproportionate to need |
| **Planning instrument** | HRPs average 34% funded vs Regional plans at 21% — same humanitarian context, different plan type, different outcome |
| **Reporting** | FTS is donor-self-reported; 3–9 month lag; non-OECD donor flows underreport; all figures are lower bounds |
| **English / UN-system** | Plans coordinated through UN agencies + English-language reporting get better-tracked outcomes |
| **Visibility** | Ukraine 65% funded vs Yemen 29% — both at maximum operational constraint (op_env 10), funding gap is a visibility outcome |
| **Timing** | Partial-year data (e.g., 2026 YTD) compared to closed years creates apparent cliffs; the deck and `country_rollup` distinguish |
| **Severity definition** | INFORM Severity Index rescaled in Jan 2026 (1–5 → 1–10); our temporal signals are scale-invariant via categorical labels |

### 5.2 What the system cannot see

- Crises without documented INFORM Severity **or** formal humanitarian plans (e.g., DPRK)
- Bilateral aid flows not reported to FTS
- Sub-national variation within country plans
- Forward-looking statistical forecasts (the system is **surveillance, not prediction**; "anticipatory action" in OCHA's formal sense means acting on early-warning signals, not statistical forecasting)

### 5.3 Data freshness

- **FTS** funding: most current at deck build (May 2026); 3–9 month reporting lag implies all figures are lower bounds
- **INFORM Severity:** monthly through April 2026 (1-month publication lag)
- **HNO PIN:** 2024 and 2025 full-year data available; 2026 partial

---

## 6. Responsible AI stance

The gap_score is a **deterministic SQL/Python pipeline**. No LLM is in the scoring path. The 53/53 verification tests confirm output stability across pipeline runs.

LLM-based components (Databricks Genie Space) operate **only on the query surface** — translating natural-language analyst questions into SQL against the verified Parquet outputs. The deterministic core IS our responsible-AI position: every score is auditable to a verifiable Parquet row, every claim cites that source, and no LLM hallucination can drift the rankings.

**Architecturally:** the deterministic core + LLM-on-query-surface is a deliberate design pattern. It gives users the natural-language ergonomics of Genie discovery *while preserving the audit trail and reproducibility that operational humanitarian decision-making requires*.

---

## 7. Decision-support discipline

The system **surfaces analytical options; the user decides.** Three layers of action surfaced by the deck:

| Layer | What it means | Source |
|---|---|---|
| **Stabilize** | Close the Tier 1 gap today ($0.6B Lebanon top-up → $7.3B full Tier 1 closure) | The gap_score Tier 1 set |
| **Pre-empt** | Act on rising-risk signals (Angola needs assessment, Mauritania plan development, watch-list monitoring) | The bonus task temporal layer |
| **Reform** | Address the structural forces — donor coordination on regional-plan parity, standardized treatment of partial-year coverage, attention to Y-o-Y volatility | The four systemic-forces analysis (donor fatigue −36%, plan-type bias, visibility bias, requirements-mask-cuts) |

The architectural principle on every dashboard page and every slide: **decision support, not decision-making.** The system points at signals — OCHA points the system at the decisions that matter.

---

## 8. What's NOT in this writeup

This is a brief writeup focused on design, architecture, and evaluation. The following are documented elsewhere:

- **Full slide deck** (anticipatory-action narrative) — submitted alongside this repo
- **Demo video** (≤15 min walkthrough of the dashboard + Genie + key findings)
- **Pipeline source** — the Databricks notebook in this repo is the canonical executed version
- **Detailed findings** (cross-cutting frames: Root cause, Urgency, Persistence, Decision trap, Decision frame, Delivery reality, Blind spot) — referenced in the slide deck; complete catalogue in the project's working documents

---

## Reproducibility

Every claim in this writeup, in the slide deck, and in the dashboard traces to a row in the Parquet outputs produced by the notebook in this repo. The 53/53 verification tests at the end of the notebook are the integrity guarantee. To re-run:

1. Open the notebook in Databricks (or a Jupyter environment with the standard data-science stack: pandas, pyspark, openpyxl)
2. Point the loaders at HDX-published source data (HNO, HRP, FTS, INFORM Severity Index, COD population — all public). **Note:** as shipped, the notebook reads these from a private Databricks Unity Catalog volume (`/Volumes/cmu_hackathon/...`) via `spark`/`dbutils`, so re-running outside that workspace requires re-pointing the loaders at local copies of the public files — the artifact is fully *auditable* as executed, but not re-runnable standalone without this step.
3. Run all stages in order
4. The verification cell at the end will report 53/53 if outputs match the documented findings

---

*This document was authored for the UN OCHA Geo-Insight Challenge submission. May 2026.*
