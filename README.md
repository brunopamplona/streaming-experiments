# streaming-experiments

> A/B experiment design with sample size calculation, power analysis, and result interpretation.  
> Includes a full geo-lift experiment with synthetic holdout region applied to a real-world streaming platform case.  
---

## Overview

This repository contains experiment design frameworks and simulated datasets for measuring the causal impact of marketing campaigns on streaming platform subscriptions. It covers two complementary methodologies:

- **A/B experimentation** — classical randomised controlled trials for product and pricing tests, with sample size calculation and power analysis utilities
- **Geo-lift experimentation** — geographic holdout studies for measuring incremental lift from paid media campaigns, where user-level randomisation is not feasible

The geo-lift case study is built around **Video+ Premiere**, a premium streaming package, and uses São Paulo (treatment) and Rio de Janeiro (control) as the two experimental markets.

---

## Repository structure

```
streaming-experiments/
│
├── data/
│   ├── weekly_subscriptions.csv        # Weekly net adds per 100k internet users (SP vs RJ, 10 weeks)
│   ├── channel_spend_impressions.csv   # Weekly spend, impressions and clicks by channel (SP only)
│   ├── user_level_conversions.csv      # Individual Premiere upgrade events with device and channel attribution
│   ├── internet_users_baseline.csv     # Population denominator by city and access segment
│   └── geo_fence_validation.csv        # Week 1 geo-fence audit — impression delivery by state
│
├── notebooks/
│   ├── ab_experiment_design.ipynb      # Sample size calculator, power curves, result interpretation
│   └── geo_lift_analysis.ipynb         # DID regression, synthetic control, interrupted time series
│
└── README.md
```

---

## Geo-lift experiment — Video Streaming Premiere Monthly Subscription Plan

### Objective

Measure the **incremental** number of Premiere plan subscriptions driven by a paid marketing campaign in São Paulo, above what would have occurred organically — using Rio de Janeiro as a synthetic holdout control.

### Why geo-lift instead of A/B?

For a streaming subscription campaign running across wallet-garden Platforms (Google, Meta, tiktok Ads, Youtube Ads) there´s no shared identity layer to randomize users on, and each platform reports lift through its own biased attribution model. A geo-lift study assigns entire cities to treatment or control and compares subscription trends between them - producing a causal lift estimate that´s independent of any ad platform's attribution.

### Market design

| City | Role | Internet users | Campaign active |
|---|---|---|---|
| São Paulo | Treatment | ~22.5 million | Yes — full budget |
| Rio de Janeiro | Control (holdout) | ~11.2 million | No — zero paid media |

Internet users are defined as individuals aged 10+ who accessed the internet in the past 3 months via any device (mobile, desktop, or tablet), sourced from IBGE 2023, ANATEL, and GSMA Mobile Economy Brazil 2024. This denominator was chosen over broadband households to capture the mobile-first majority in both cities.

### Experiment timeline

| Period | Weeks | Description |
|---|---|---|
| Pre-period baseline | −2 to 0 | Both cities dark — no paid Premiere media. Establishes parallel trend and organic baseline. |
| Soft launch | 1 | SP activated at 20% budget. Geo-fence accuracy validated. RJ sub rate monitored for spillover. |
| Full flight | 2–6 | 100% budget live in SP across five channels. RJ remains dark. |
| Wind-down | 7 | SP spend drops to zero. Post-flight conversion tail begins. |
| Post-flight observation | 8 | Both cities tracked. Full conversion window captured. |
| Analysis and readout | 9–10 | DID regression, synthetic control, ITS validation, iROAS calculation. |

### Campaign channels (São Paulo only)

| Channel | Spend share | Geo targeting method |
|---|---|---|
| Digital video (Google Ads) | 25% | City = São Paulo, RJ excluded |
| Digital video (YouTube Ads) | 25% | City = São Paulo, RJ excluded |
| Digital video (Meta Ads) | 25% | City = São Paulo, RJ excluded |
| Digital video (TikTok Ads) | 26% | City = São Paulo, RJ excluded |

### Measurement methodology

**Primary — difference-in-differences (DID)**

```
Lift = (SP_post − SP_pre) − (RJ_post − RJ_pre)
```

Both cities are indexed to net new Premiere subscriptions per 100k internet users per week. The pre-period establishes the counterfactual trend; the post-period difference is attributed to the campaign.

**Secondary — interrupted time series (ITS)**

São Paulo's sub rate is modelled as a segmented regression with a structural break at campaign launch week. The slope change after the break is the causal lift estimate. Useful with only two markets, as it extracts signal from SP's own pre/post trajectory independently of RJ.

### Key output metrics

| Metric | Formula | Target |
|---|---|---|
| Incremental Premiere net adds | SP lift − RJ counterfactual trend | p < 0.10 |
| Incremental cost per sub (iCPS) | Total SP spend ÷ incremental net adds | Below LTV/3 payback |
| iROAS | (Incremental adds × 12-mo LTV) ÷ SP spend | > 2.0× |
| Cannibalization rate | Drop in lower-tier subs in SP vs RJ | < 15% of gross adds |
| Mobile vs desktop lift split | Net adds by device type SP vs RJ | Directional |

### Design considerations for a 2-market experiment

With only two markets, statistical power comes from within-market time-series variance across the 10-week window rather than cross-market variance. The minimum detectable lift is approximately **12%** at 80% power and α = 0.10. Key risks to manage:

- **SP→RJ social spillover** — influencer and social content does not respect city borders. RJ sub rate is monitored weekly; a >8% anomalous rise above its own pre-period baseline triggers a review.
- **Carnival seasonality** — Rio de Janeiro's Carnival drives a distinct organic streaming spike not mirrored in SP. Schedule the experiment to avoid overlap, or model the spike as a covariate in the DID regression.
- **No holdout redundancy** — with a single control market, any idiosyncratic shock to RJ (local news event, competitor outage) has no fallback comparator. All RJ-specific events during the flight should be documented for post-hoc adjustment.
- **Geo-fence leakage** — the Week 1 soft launch audit (`geo_fence_validation.csv`) revealed that 0.9% of impressions were served to RJ users. Targeting was tightened before full flight in Week 2.

---

## A/B experiment design

The `ab_experiment_design.ipynb` notebook covers:

- **Sample size calculation** using the two-proportion z-test for subscription conversion rate experiments
- **Power analysis** with interactive power curves across a range of minimum detectable effects (MDE)
- **Runtime estimation** based on daily traffic and allocation ratio
- **Result interpretation** — statistical significance, practical significance, confidence intervals, and guardrail metrics
- **Common pitfalls** — peeking, multiple comparisons, novelty effects, and network interference

---

## Data dictionary

### `weekly_subscriptions.csv`

| Column | Description |
|---|---|
| `week` | Week index (−2 to 8; negative = pre-period) |
| `phase` | Human-readable experiment phase label |
| `city_sp_net_adds_per100k` | SP net new Premiere subs per 100k internet users |
| `city_rj_net_adds_per100k` | RJ net new Premiere subs per 100k internet users |
| `sp_campaign_lift_applied` | Simulated campaign lift added to SP organic baseline |
| `rj_spillover_applied` | Simulated organic spillover reaching RJ (3–6% of SP lift) |

### `channel_spend_impressions.csv`

| Column | Description |
|---|---|
| `week` | Week index |
| `channel` | Marketing channel name |
| `spend_brl` | Weekly spend in Brazilian reais |
| `impressions` | Total impressions served |
| `clicks` | Total clicks |
| `ctr_pct` | Click-through rate (%) |
| `city` | Always SP — campaign ran in treatment market only |

### `user_level_conversions.csv`

| Column | Description |
|---|---|
| `user_id` | Anonymised user identifier |
| `conversion_date` | Date of Premiere upgrade |
| `device_type` | mobile / desktop / tablet |
| `previous_plan` | Plan before upgrading (free / ad-supported / basic) |
| `last_touch_channel` | Last marketing touchpoint before conversion |
| `days_to_convert` | Days between first ad exposure and conversion |
| `premiere_monthly_brl` | Monthly plan value at time of conversion (R$ 34.90) |

### `internet_users_baseline.csv`

Population denominator reference for SP and RJ, segmented by internet access type (mobile only, mobile + broadband, broadband only, other). Source: IBGE 2023 / ANATEL / GSMA 2024.

### `geo_fence_validation.csv`

Week 1 impression delivery audit by Brazilian state. Flags any impressions served outside the SP target region and documents the required action (none / monitor / tighten fence).

---

## Requirements

```
python >= 3.10
pandas
numpy
scipy
statsmodels
matplotlib
```

---

## Author

**Bruno Pamplona**  
Experiment design · causal inference · streaming analytics
