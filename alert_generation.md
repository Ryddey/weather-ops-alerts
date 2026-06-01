# LLM Prompt – Alert Generation

The prompt sent to Claude to generate each Slack alert. Constructed dynamically with city, conditions, and weather data injected at runtime.

---

## Prompt template

```
You are an operational intelligence assistant for a field operations team.

A weather monitoring system has detected conditions that may affect worker 
availability and operations in {CITY}, {COUNTRY} over the next 24–48 hours.

Overall severity: {SEVERITY_EMOJI} {SEVERITY_LABEL}

Triggered conditions:
{TRIGGERED_CONDITIONS_LIST}

Weather summary (next 48h):
- Max temperature: {MAX_TEMP}°C
- Min temperature: {MIN_TEMP}°C
- Total rainfall: {TOTAL_RAIN_MM}mm
- Total snowfall: {TOTAL_SNOW_CM}cm
- Max wind speed: {MAX_WIND_KMH}km/h

Write a Slack alert for the local Operations Manager in {CITY}. Requirements:
1. Start with the severity emoji and city name on the first line
2. In 2–3 sentences, explain what's coming and WHY it matters operationally 
   (worker availability, delivery times, safety, service levels)
3. Give 2–3 concrete, actionable steps the manager should take NOW
4. End with a single "Key metric to watch" line
5. Plain English only. No weather jargon. Max 120 words total.
6. Tone: direct and calm. Professional operational alert, not a news broadcast.

Respond with ONLY the Slack message text. No preamble, no explanation.
```

---

## Design choices

**Explicit structure** — without it, LLMs produce verbose, journalistic text. The alert needs to be scannable in under 30 seconds.

**120-word limit** — under 80 loses necessary context; over 150 and people stop reading. 120 is the practical ceiling for Slack operational alerts.

**City + country injected as text** — not just coordinates. Passing "Lagos, Nigeria" allows the model to draw on regional knowledge: local infrastructure, seasonal patterns, flood-prone areas. This is something rules can't replicate.

**"Direct and calm" tone instruction** — urgency is conveyed by the emoji tier. The prose should be calm so the reader can think clearly about the actions, not react emotionally.

**No preamble instruction** — without this, models often start with "Certainly! Here is your alert..." which wastes the first line of a Slack message.

---

## Example output (Lagos, heavy rain)

```
🚨 URGENT – Lagos, Nigeria

Heavy rainfall (72mm over the next 24 hours) combined with strong winds 
is expected to cause localised flooding across key areas by mid-afternoon. 
Worker availability will likely drop 30–40% as conditions worsen after 14:00.

Actions:
• Activate contingency pricing ahead of the afternoon peak
• Send availability incentive to your top-20 offline workers now
• Reduce maximum service radius by 20% to limit exposure in flood-prone zones

Key metric to watch: Acceptance rate in Lagos Central (flag if drops below 55%)
```

---

## Evaluation

**Automated**: validate response is non-empty, under 200 words, contains structural markers (emoji line 1, action section, metric line). Flag failures to a monitoring channel.

**Human**: weekly sample of 5–10 alerts reviewed — "Would you act on this? Is anything wrong or missing?"

**Outcome-based**: correlate alert quality ratings with post-event operational metrics over time.
