# Project Charter — PS Resource Intelligence Hub

> **Version:** 1.0 | **Date:** July 2026 | **Status:** Active Build
> **Author:** Resource & Operations Manager

---

## 1. Executive Summary

Professional Services teams lose revenue and miss delivery timelines because resourcing,
staffing, and forecasting are managed through disconnected spreadsheets, manual updates,
and reactive communication.

This project builds an automation-first forecasting and staffing system that:
- Pulls demand and supply data from Salesforce (or Google Sheets as interim source)
- Calculates 4 KPIs automatically every week via n8n
- Uses Claude AI to produce structured, executive-ready forecast summaries
- Delivers outputs to Google Sheets and Gmail with zero manual effort

---

## 2. Problem Statement

| Pain Point | Business Impact |
|---|---|
| Staffing decisions made reactively | Projects start understaffed — SLA risk |
| Forecast data in siloed spreadsheets | No single source of truth — low accuracy |
| Manual stakeholder updates | Slow leadership visibility — escalation risk |
| No automated gap alerting | Revenue at risk goes unnoticed |
| Utilization tracked by hand | Burnout and bench waste both go undetected |

---

## 3. KPI Priority Order

| # | KPI | Formula | Alert Threshold |
|---|---|---|---|
| 1 | Fill Rate | Filled Roles / Total Required Roles × 100 | < 70% — send alert |
| 2 | Forecast Accuracy Delta | (Planned Hrs − Available Hrs) / Planned Hrs × 100 | > 20% — flag |
| 3 | Utilization Rate | Assigned Hrs / Available Capacity Hrs × 100 | < 60% or > 90% |
| 4 | Revenue at Risk | Unfilled Required Hrs × Avg Billing Rate | > $100K — send alert |

---

## 4. Tech Stack

| Layer | Tool | Why |
|---|---|---|
| Source (primary) | Salesforce Developer Edition | Real PSA-style objects, free forever |
| Source (MVP interim) | Google Sheets | Fast to set up, identical data structure |
| Orchestration | n8n | Native Salesforce + Sheets + Gmail nodes, open source |
| AI Insight Layer | Claude API (Anthropic) | Structured JSON outputs, best narrative quality |
| Output — Data | Google Sheets | Forecast log + current state tabs |
| Output — Alerts | Gmail | Conditional HTML email on threshold breach |

---

## 5. Scope

### In Scope (v1)
- Scheduled data pull from Salesforce or Google Sheets
- Demand vs. supply role-matching logic (role + region + availability date)
- Weekly auto-calculation of 4 KPIs
- Claude structured JSON forecast summary
- Google Sheets write: forecast_log + current_state tabs
- Conditional Gmail alert (HTML, only when threshold breached)

### Out of Scope (v1)
- Bidirectional Salesforce writeback
- ML forecasting model
- Real-time streaming data
- Production-grade security packaging

---

## 6. Architecture

```
[n8n Schedule Trigger — Every Monday 8:00 AM CST]
        |
        v
[Read demand_data]          [Read supply_data]
(Sheets or Salesforce)      (Sheets or Salesforce)
        |                          |
        +----------+---------------+
                   |
                   v
         [KPI Calculator — n8n Code Node (JavaScript)]
         - Match roles: role_needed = resource.role AND region match
           AND resource.available_from <= project.start_date
         - Calculate: Fill Rate, Unfilled Roles, Revenue at Risk,
           Forecast Accuracy Delta
         - Build demand_summary text block for Claude
         - Build supply_summary text block for Claude
                   |
                   v
         [Claude API — HTTP Request Node]
         Model: claude-sonnet-4-5
         Output: Structured JSON per claude_forecast_output.json schema
                   |
          +--------+----------+
          |                   |
          v                   v
  [IF alert_required]  [Write forecast_log tab — append row]
      |                [Write current_state tab — overwrite]
      v
  [Send Gmail alert — HTML formatted]
```

---

## 7. Data Model

### demand_data (Google Sheets tab OR Salesforce object)

| Field | Type | Notes |
|---|---|---|
| project_id | String | PRJ-001 format |
| project_name | String | |
| client | String | |
| region | String | LATAM / NA / EMEA |
| role_needed | String | Must exactly match supply.role for fill matching |
| start_date | Date | YYYY-MM-DD |
| end_date | Date | YYYY-MM-DD |
| required_hours | Integer | Total hours needed for this role |
| stage | String | Confirmed / Proposed |
| priority | String | High / Medium / Low |
| billing_rate_usd | Integer | Hourly rate for Revenue at Risk calc |

### supply_data (Google Sheets tab OR Salesforce object)

| Field | Type | Notes |
|---|---|---|
| resource_id | String | RES-001 format |
| name | String | |
| role | String | Must match demand.role_needed |
| skills | String | Comma-separated |
| region | String | LATAM / NA / EMEA |
| utilization_pct | Integer | 0–100 |
| available_from | Date | Earliest new-assignment date |
| bench_status | String | Full Bench / Partial / Rolling Off / Unavailable |
| seniority | String | Senior / Mid / Junior |

### forecast_log (n8n writes here — append mode)

| Field | Type | Notes |
|---|---|---|
| run_date | Date | Auto-set by n8n |
| total_demand_roles | Integer | |
| filled_roles | Integer | |
| unfilled_roles | Integer | |
| fill_rate_pct | Decimal | |
| total_required_hours | Integer | |
| covered_hours | Integer | |
| uncovered_hours | Integer | |
| revenue_at_risk_usd | Decimal | |
| forecast_accuracy_delta_pct | Decimal | |
| claude_summary | String | Executive summary from Claude |
| alert_sent | String | YES / NO |
| alert_reason | String | Threshold description |

---

## 8. 4-Week Roadmap

| Week | Milestone | Done When |
|---|---|---|
| Week 1 | Data layer + KPI engine | Google Sheets source live; n8n calculates all 4 KPIs |
| Week 2 | Claude + Gmail | Claude returns valid JSON; Gmail fires on threshold |
| Week 3 | Salesforce source | n8n reads Salesforce Dev Org via OAuth |
| Week 4 | Polish + publish | Architecture diagram, README, screenshots, public GitHub |

---

## 9. Success Criteria

- [ ] System runs end-to-end with no manual steps
- [ ] All 4 KPIs auto-calculated correctly
- [ ] Claude summary is valid JSON matching schema every run
- [ ] Gmail alert fires on threshold breach
- [ ] Forecast Log has at least 4 weekly entries
- [ ] GitHub repo is public with working README and setup guide

---

## 10. Alignment with ServiceNow / CoreX

| This Project | ServiceNow Equivalent |
|---|---|
| demand_data | Resource Plans / Project Demands |
| supply_data | Resource Pools / Worker Profiles |
| Fill Rate KPI | Resource Fulfillment Rate |
| Claude forecast summary | Demand Forecast view |
| Gmail alert | ServiceNow threshold notification |
| n8n orchestration | Flow Designer / Integration Hub |

This project is architecturally aligned with how ServiceNow Resource Management and
Workforce Optimization handle demand forecasting — making it directly relevant to
Global Resource Manager roles in ServiceNow-based delivery organizations.

---

*Last updated: July 8, 2026*
