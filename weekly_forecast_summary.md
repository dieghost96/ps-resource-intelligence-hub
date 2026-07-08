# Weekly Forecast Summary — Claude Prompt
## `prompts/weekly_forecast_summary.md`

This prompt is sent to Claude every Monday at 8 AM via n8n.
n8n does the math first. Claude interprets, prioritizes, and communicates.

---

## How to wire this in n8n

1. Add an **HTTP Request node** after your KPI Calculator node
2. Method: `POST`
3. URL: `https://api.anthropic.com/v1/messages`
4. Headers:
   - `x-api-key` → your Anthropic API key (from n8n credential: HTTP Header Auth)
   - `anthropic-version` → `2023-06-01`
   - `content-type` → `application/json`
5. Body (JSON — see template below, replace `{{ }}` with n8n expressions)

---

## SYSTEM MESSAGE

Paste this text as the value of the `"system"` field in your n8n HTTP Request body:

```
You are a Professional Services Resource Intelligence assistant. Your role is to analyze
weekly staffing and forecasting data for a delivery organization and produce concise,
executive-ready summaries.

You will receive:
- Pre-calculated KPIs (fill rate, unfilled roles, revenue at risk, forecast accuracy delta)
- Demand data: open roles, required hours, start dates, stages, billing rates
- Supply data: available resources, skills, regions, utilization, bench status

Your output MUST be valid JSON only — no markdown, no explanation text, just the JSON.
Match this exact structure:

{
  "run_date": "YYYY-MM-DD",
  "kpis": {
    "fill_rate_pct": number,
    "unfilled_roles": integer,
    "revenue_at_risk_usd": number,
    "forecast_accuracy_delta_pct": number
  },
  "top_risks": [
    {
      "rank": 1,
      "project_id": "string",
      "project_name": "string",
      "risk_type": "Unstaffed Role | Over-allocation | Forecast Drift | Revenue at Risk | Start Date Conflict",
      "risk_description": "string (max 200 chars)",
      "recommended_action": "string (max 200 chars)",
      "severity": "Critical | High | Medium | Low"
    }
  ],
  "executive_summary": "string (2-3 sentences, max 400 chars, no jargon)",
  "alert_required": true or false,
  "alert_reason": "string or empty string"
}

Rules:
- Do not invent data. Base everything on what was provided.
- Tone: Direct, professional. Write like a Chief of Staff briefing a VP.
- recommended_action should name a specific resource (by name and ID) when a match exists.
```

---

## USER MESSAGE TEMPLATE

Paste this as the value of `"content"` inside `"messages"`.
Replace each `{{ expression }}` with the actual n8n expression:

```
## Weekly Resource Forecast — {{ $json.run_date }}

### Pre-Calculated KPIs (do not recalculate — use these exact values)
- Fill Rate: {{ $json.fill_rate_pct }}%
- Unfilled Roles: {{ $json.unfilled_roles }}
- Revenue at Risk: ${{ $json.revenue_at_risk_usd }}
- Forecast Accuracy Delta: {{ $json.forecast_accuracy_delta_pct }}%

### Open Demand — next 90 days (one line per required role)
{{ $json.demand_summary }}

Example of what n8n builds:
PRJ-001 | Digital Transformation | Acme Corp | LATAM | Senior Consultant | Start: Aug 1 | 480 hrs | Confirmed | $72,000

### Available Supply (one line per resource)
{{ $json.supply_summary }}

Example of what n8n builds:
RES-001 | Alex Torres | Senior Consultant | LATAM | 40% utilized | Available: Jul 15 | Partial Bench | Senior

### Instructions
1. Identify the top 1-3 staffing or forecast risks, ranked by business impact
2. For each risk, provide a specific recommended action — name the best matching resource if one exists
3. Write a 2-3 sentence executive_summary a Resource Manager could send directly to a VP
4. Set alert_required = true if fill_rate_pct < 70 OR revenue_at_risk_usd > 100000

Return ONLY valid JSON. No markdown. No text before or after the JSON.
```

---

## Full n8n Body (copy-paste ready, swap expressions)

```json
{
  "model": "claude-sonnet-4-5",
  "max_tokens": 1024,
  "system": "PASTE SYSTEM MESSAGE HERE",
  "messages": [
    {
      "role": "user",
      "content": "PASTE USER MESSAGE TEMPLATE HERE WITH n8n EXPRESSIONS"
    }
  ]
}
```

---

## Alert Thresholds

| KPI | Trigger `alert_required = true` |
|-----|----------------------------------|
| Fill Rate | < 70% |
| Revenue at Risk | > $100,000 |
| Both thresholds breached | Alert still = single email |

---

## Example Valid Claude Output

```json
{
  "run_date": "2026-08-03",
  "kpis": {
    "fill_rate_pct": 62.5,
    "unfilled_roles": 3,
    "revenue_at_risk_usd": 163800,
    "forecast_accuracy_delta_pct": 14.2
  },
  "top_risks": [
    {
      "rank": 1,
      "project_id": "PRJ-002",
      "project_name": "ERP Core Implementation",
      "risk_type": "Unstaffed Role",
      "risk_description": "Project Manager required for Aug 15 start. 640 hours, $112K at risk. No confirmed assignment.",
      "recommended_action": "Assign RES-008 (Chloe Dubois, Senior PM, NA, 30% utilized, available Aug 10). Confirm by EOW.",
      "severity": "Critical"
    },
    {
      "rank": 2,
      "project_id": "PRJ-005",
      "project_name": "Cloud Infrastructure Audit",
      "risk_type": "Unstaffed Role",
      "risk_description": "Solutions Architect needed Sep 15. RES-005 rolling off but 80% utilized through Sep 1.",
      "recommended_action": "Confirm RES-005 availability post Sep 1. If unavailable, open external sourcing immediately.",
      "severity": "High"
    },
    {
      "rank": 3,
      "project_id": "PRJ-007",
      "project_name": "Analytics Dashboard Build",
      "risk_type": "Forecast Drift",
      "risk_description": "Proposed stage, Oct 1 start, no committed Data Engineer in LATAM.",
      "recommended_action": "Monitor through Aug 15. Escalate if stage moves to Confirmed.",
      "severity": "Low"
    }
  ],
  "executive_summary": "Fill rate is at 62.5% with 3 unstaffed roles and $163.8K revenue at risk. PRJ-002 (TechCo ERP, Aug 15 start) is critical — RES-008 is the strongest match and should be confirmed by end of week. PRJ-005 requires sourcing action before Sep 1 to avoid a deployment gap.",
  "alert_required": true,
  "alert_reason": "Fill Rate 62.5% below 70% threshold. Revenue at Risk $163,800 above $100,000 threshold."
}
```
