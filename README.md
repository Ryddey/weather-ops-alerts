# 🌩️ Weather Ops Alert System

An automated pipeline that monitors weather forecasts for cities across different climate zones and posts structured operational alerts to Slack when conditions are likely to disrupt courier or field operations.

Built to explore the intersection of weather data, rule-based automation, and LLM-generated operational guidance.

---

## What it does

1. Pulls 48-hour weather forecasts every 3 hours via [Open-Meteo](https://open-meteo.com/) (free, no API key)
2. Evaluates conditions against city-relative thresholds (not universal ones)
3. Uses Claude to turn raw weather signals into plain-language, actionable Slack alerts
4. Deduplicates alerts so the same event doesn't spam the channel

---

## Cities monitored

| City | Country | Climate baseline |
|------|---------|-----------------|
| Tallinn | Estonia | Cold winters, snow common |
| Madrid | Spain | Hot and dry, rare snow |
| Warsaw | Poland | Mixed continental |
| Lagos | Nigeria | Tropical, heavy rain season |

The mix is intentional — it forces the threshold logic to be **climate-relative**. 5cm of snow is unremarkable in Tallinn and extraordinary in Lagos.

---

## Architecture

```
Open-Meteo API (x4 cities)
        │
        ▼
  n8n Schedule (every 3h)
        │
        ▼
  Fetch & parse forecasts
        │
        ▼
  City-specific threshold evaluation
        │
  ┌─────┴──────────┐
  │ No alert       │ Alert triggered
  │ (exit quietly) │
  └────────────────┘
        │
        ▼
  LLM (Claude) — generate alert text
  combining multiple signals into
  plain-language operational guidance
        │
        ▼
  Deduplication (6h window)
        │
        ▼
  Slack Webhook → #ops-weather-alerts
```

---

## Key design decisions

**Climate-relative thresholds**
Rather than one global threshold per condition, each city has its own. The trigger point is "what exceeds what local infrastructure and workers are prepared for", not an arbitrary absolute value.

**Two alert tiers**
- `⚠️ Heads-up` — conditions are building, plan ahead
- `🚨 Urgent` — conditions are severe, act now

**LLM for alert text, not rules**
Rules identify *which* conditions are triggered. Claude synthesises them into a coherent, city-specific narrative: what's coming, why it matters operationally, and what to do. This handles compound conditions (rain + wind + timing) that a template can't.

**Deduplication without external infrastructure**
Uses n8n's static workflow data as a lightweight key store. Same city + same condition combination won't alert again within a 6-hour block. No Redis needed for a self-hosted setup.

---

## Setup

### Prerequisites

- n8n running locally (`npx n8n` or Docker)
- A Slack workspace with an [Incoming Webhook](https://api.slack.com/messaging/webhooks)
- A Claude API key ([console.anthropic.com](https://console.anthropic.com))

### Run n8n

```bash
# Via npx
npx n8n

# Via Docker
docker run -it --rm -p 5678:5678 n8nio/n8n
```

Open `http://localhost:5678`

### Import the workflow

1. Go to **Workflows → Import from file**
2. Select `n8n/weather_alerts.json`
3. Add your credentials:

| Field | Where | Value |
|---|---|---|
| Slack URL | HTTP Request node → URL | Your Incoming Webhook URL |
| Anthropic key | HTTP Request node → Header | Your Claude API key |

### Activate

Toggle the workflow to **Active**. It runs every 3 hours automatically.

---

## Repo structure

```
weather-ops-alerts/
├── README.md
├── n8n/
│   └── weather_alerts.json     ← Import this into n8n
├── docs/
│   └── design-notes.md         ← Threshold logic and design rationale
└── prompts/
    └── alert_generation.md     ← The LLM prompt with commentary
```

---

## What I'd build next

- **Supply prediction layer** — use weather forecasts + historical data to predict worker availability drops before they happen, and trigger proactive incentives
- **Feedback loop** — a Slack button on each alert (✅ Useful / ❌ Noise) to evaluate alert quality over time
- **Outcome correlation** — connect alerts to downstream operational metrics to validate that threshold choices are actually predictive
