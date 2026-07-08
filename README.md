# PS Resource Intelligence Hub

> An automation-first demand forecasting and staffing system for Professional Services
> Salesforce · n8n · Claude AI · Google Sheets · Gmail

<img width="671" height="751" alt="image" src="https://github.com/user-attachments/assets/da4448e4-2f16-4611-a43c-b3376df67f00" />


---

## What This Does

Every Monday at 8 AM, this system automatically:

1. Pulls demand and supply data from Salesforce (or Google Sheets as interim source)
2. Calculates 4 core KPIs: Fill Rate, Forecast Accuracy, Utilization, Revenue at Risk
3. Sends data to Claude AI and receives a structured JSON forecast summary
4. Writes results to a Google Sheets Forecast Log (running history)
5. Sends a conditional Gmail alert if Fill Rate < 70% or Revenue at Risk > $100K

No manual work. No spreadsheet gymnastics. Decisions before problems.

<img width="1727" height="657" alt="image" src="https://github.com/user-attachments/assets/9cb3d74e-cb27-4a86-9163-7f2f875d08bc" />


---

## Architecture

```
[n8n Schedule]  -->  [Read Salesforce / Sheets]  -->  [KPI Calculator (Code Node)]
                                                               |
                                                        [Claude API]
                                                      structured JSON
                                                               |
                                          +-----------------+--+------------------+
                                          |                                       |
                               [Google Sheets Log]                  [Gmail Alert if threshold]
```

Full detail: docs/01-project-charter.md

---

## KPIs Tracked

| # | KPI | Formula | Alert Threshold |
|---|---|---|---|
| 1 | Fill Rate | Filled Roles / Total Required Roles x 100 | < 70% |
| 2 | Forecast Accuracy Delta | (Planned - Available Hours) / Planned x 100 | > 20% |
| 3 | Utilization Rate | Assigned Hours / Available Capacity x 100 | < 60% or > 90% |
| 4 | Revenue at Risk | Unfilled Hours x Avg Billing Rate | > $100,000 |

---

## Tech Stack

| Layer | Tool |
|---|---|
| Source (primary) | Salesforce Developer Edition |
| Source (MVP interim) | Google Sheets |
| Orchestration | n8n |
| AI Insight Layer | Claude API (claude-sonnet-4-5) |
| Output — Data | Google Sheets |
| Output — Alerts | Gmail |

---

## Repo Structure

```
ps-resource-intelligence-hub/
├── README.md
├── .env.example                              <- API keys template (never commit .env)
├── docs/
│   └── 01-project-charter.md                <- Full definition, KPIs, architecture, roadmap
├── workflows/
│   └── n8n/
│       └── forecast_alert_v1.json           <- Importable n8n workflow (added Week 2)
├── schemas/
│   └── claude_forecast_output.json          <- JSON schema for Claude structured output
├── sample-data/
│   ├── demand_data.csv                      <- 8 sample projects (roles, billing rates, dates)
│   ├── supply_data.csv                      <- 9 sample resources (skills, availability)
│   └── forecast_log_template.csv           <- Google Sheets log tab — headers + example row
└── prompts/
    └── weekly_forecast_summary.md          <- Claude prompt + n8n variable mapping guide
```

---

## Setup Guide

### Prerequisites

- n8n account (app.n8n.cloud — free cloud tier works)
- Anthropic API key (console.anthropic.com — free credits available)
- Google account (for Sheets + Gmail)
- Salesforce Developer Edition (developer.salesforce.com/signup — free forever)

---

### Step 1 — Google Sheets (15 min)

What you are doing: Creating the data source and output destination before Salesforce is connected.

1. Create a new Google Sheet named: PS Resource Intelligence Hub
2. Add 4 tabs: demand_data | supply_data | forecast_log | current_state
3. Paste demand_data.csv (headers + all rows) into the demand_data tab
4. Paste supply_data.csv (headers + all rows) into the supply_data tab
5. Paste only the header row from forecast_log_template.csv into the forecast_log tab
6. Leave current_state empty — n8n will populate it on each run
7. Copy your Sheet ID from the URL: docs.google.com/spreadsheets/d/YOUR_ID/edit

---

### Step 2 — Connect n8n to Google Sheets (10 min)

What you are doing: Giving n8n read/write access to your Sheet.

1. n8n -> Credentials -> Add Credential -> Google Sheets OAuth2
2. Sign in with your Google account and grant access
3. Test: create a Sheets node, enter your Sheet ID, read demand_data tab
4. If rows appear, the connection works

---

### Step 3 — Build the n8n Forecast Workflow (Week 1)

What you are doing: Building the core logic engine — one node per job.

Nodes in order:
1. Schedule Trigger — Every Monday 8:00 AM CST
2. Read Demand — Google Sheets node, demand_data tab
3. Read Supply — Google Sheets node, supply_data tab
4. KPI Calculator — Code node (JavaScript):
   - Match roles: role_needed = resource.role AND region match AND available_from <= start_date
   - Calculate: Fill Rate, Unfilled Roles, Revenue at Risk, Forecast Accuracy Delta
   - Build demand_summary and supply_summary text blocks for Claude
5. Claude — HTTP Request to Anthropic API (see prompts/weekly_forecast_summary.md)
6. Parse Claude Response — JSON parse node
7. IF Alert — condition: alert_required == true
8a. Send Gmail — conditional HTML alert (if true)
8b. Continue — no email (if false)
9. Write forecast_log — Google Sheets append row
10. Write current_state — Google Sheets overwrite tab

The exportable workflow file will be added as workflows/n8n/forecast_alert_v1.json in Week 2.

---

### Step 4 — Add Claude API (10 min)

What you are doing: Connecting Claude so it generates the narrative summary.

1. Get API key at console.anthropic.com
2. n8n -> Credentials -> HTTP Header Auth
   - Header name: x-api-key
   - Value: your API key
3. See full prompt and request body configuration in prompts/weekly_forecast_summary.md
4. Validate Claude's response against schemas/claude_forecast_output.json

---

### Step 5 — Connect Salesforce Developer Edition (Week 3)

What you are doing: Replacing Google Sheets with a real Salesforce connection.
The rest of the workflow stays the same — only the source nodes change.

1. In Salesforce Dev Org: Setup -> App Manager -> New Connected App
   - Enable OAuth
   - Callback URL: your n8n instance URL
   - Scopes: api, refresh_token
2. n8n -> Credentials -> Salesforce OAuth2
3. Replace the two Sheets source nodes with Salesforce query nodes
4. Map Salesforce field API names to the same variable names the KPI Calculator uses

---

## Environment Variables

Copy .env.example to .env, fill in your values. Never commit .env to GitHub.

```
ANTHROPIC_API_KEY=your_key_here
GOOGLE_SHEET_ID=your_sheet_id_here
GMAIL_RECIPIENT=your_email@example.com
SALESFORCE_CLIENT_ID=your_connected_app_consumer_key
SALESFORCE_CLIENT_SECRET=your_connected_app_consumer_secret
SALESFORCE_INSTANCE_URL=https://yourorg.my.salesforce.com
```

---

## Sample Data

The sample-data/ folder contains 8 demand records and 9 supply records across LATAM, NA,
and EMEA — modeled on a real Professional Services delivery portfolio.

Intentionally calibrated to produce:
- Fill Rate 62.5% (below 70% threshold) -> Gmail alert fires
- Revenue at Risk $163.8K (above $100K threshold) -> Gmail alert fires
- 3 clear staffing mismatches -> Claude surfaces specific, named recommendations

This makes the demo immediately meaningful and every output verifiable against the raw data.

---

## Build Roadmap

- [x] Project charter and data model defined
- [x] Sample data created (demand, supply, forecast log)
- [x] Claude structured output schema designed
- [x] Claude prompt with n8n variable mapping documented
- [ ] Week 1: n8n workflow with Google Sheets source and KPI engine
- [ ] Week 2: Claude integration and conditional Gmail alerts live
- [ ] Week 3: Salesforce Developer Edition connected as source
- [ ] Week 4: Architecture diagram, README polish, public GitHub

---

## Alignment with ServiceNow / CoreX

| This Project | ServiceNow Equivalent |
|---|---|
| demand_data | Resource Plans / Project Demands |
| supply_data | Resource Pools / Worker Profiles |
| Fill Rate KPI | Resource Fulfillment Rate |
| Claude forecast summary | Demand Forecast report |
| Gmail alert | ServiceNow threshold notification |
| n8n orchestration | Flow Designer / Integration Hub |

Built to demonstrate applied AI, workflow automation, and Professional Services
operational knowledge — including direct alignment with ServiceNow-based resource
management patterns used in enterprise delivery organizations.

---

## License

MIT
