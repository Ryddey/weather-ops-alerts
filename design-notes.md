# Design Notes – Weather Ops Alert System

---

## Why city-relative thresholds?

The most common mistake in weather alerting systems is using universal thresholds. A single cutoff like "alert if snow > 5cm" sounds reasonable until you realise that 5cm in a Nordic city is a Tuesday, while the same amount in a tropical city could paralyse infrastructure.

The threshold should reflect **how far conditions deviate from what local workers and infrastructure are equipped to handle** — not an absolute meteorological value.

### Threshold table (next 48h aggregated)

| Condition | Tallinn | Madrid | Warsaw | Lagos |
|---|---|---|---|---|
| Snow (cm) — urgent | >20 | >2 | >15 | >0.1 (any) |
| Rain (mm) — urgent | >30 | >40 | >35 | >60 |
| Wind (km/h) — urgent | >65 | >70 | >65 | >55 |
| Max temp (°C) — urgent | — | >40 | >37 | >42 |
| Min temp (°C) — urgent | <-20 | <-5 | <-18 | — |

Each condition also has a lower `heads-up` threshold (roughly 60–70% of the urgent value) that triggers a lower-severity alert.

---

## Why two severity tiers?

Alert fatigue is a real failure mode. If everything is urgent, nothing is.

- **Heads-up** means: conditions are building, use this time to prepare
- **Urgent** means: conditions are imminent or already materialising, act now

The difference shows up in the Slack message tone, the emoji, and the types of actions recommended.

---

## Alert format rationale

Target: readable and actionable in under 30 seconds.

Structure:
1. Severity emoji + city name (scannable at a glance)
2. 2–3 sentences on what's coming and *why it matters operationally* (not just "it will rain")
3. 2–3 concrete actions (specific, not generic)
4. One metric to watch (closes the loop)

Word limit: ~120 words. Under 80 loses necessary context. Over 150 and people stop reading fully.

---

## Why use an LLM for alert text?

Rules can identify *which* conditions are triggered and *at what severity*. They can't do this well:

- **Compound conditions**: "30mm rain" is different from "30mm rain + 60km/h wind on a Friday afternoon". A template can't reason about combinations; an LLM can weigh them naturally.
- **City-specific narrative**: An alert for Lagos should mention flooding risk and moto-courier density. An alert for Tallinn should mention black ice and reduced daylight hours. That requires local knowledge, not string interpolation.
- **Operational translation**: "72mm precipitation" needs to become "heavy rain likely to cause flooding that will cut courier availability by ~30%". That's a language and reasoning task.

The alternative — a template per city per condition combination — doesn't scale and produces robotic text that people learn to ignore.

---

## Deduplication design

A 6-hour deduplication window keyed on `{city}_{condition_types}_{date}_{6h_block}`.

The same city + same triggered conditions won't alert again within the same 6-hour window. A *new* condition type for the same city (e.g. wind escalates to urgent when it was previously heads-up) generates a new fingerprint and does alert.

Implementation uses n8n's static workflow data — no external database required for a self-hosted setup. In production, this would move to Redis or a lightweight DB for persistence across restarts.

---

## Failure mode: silent null data

The most dangerous failure isn't a crash — it's valid JSON with null field values (which Open-Meteo occasionally returns for edge cases or during maintenance windows).

Mitigation: the threshold evaluation step filters nulls before any comparison. If all conditions evaluate to null, `should_alert = false` and the flow exits without error and without a false all-clear.

What I'd add in production: a daily heartbeat check that verifies weather data was non-null for all cities. Failure posts to a monitoring channel.

---

## Evaluation: how to know the LLM step isn't breaking quietly

1. **Structural validation**: check that the response is non-empty, under 200 words, and contains expected markers (emoji on line 1, action items, metric line). Log failures.
2. **Human spot-checks**: weekly sample of generated alerts reviewed by someone familiar with the operational context — would you act on this? Is anything missing?
3. **Outcome correlation** (longer-term): connect alert quality scores to downstream operational data. If highly-rated alerts correlate with better outcomes during weather events, the system is calibrated correctly.
