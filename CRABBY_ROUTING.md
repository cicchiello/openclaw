# Crabby — Prompt Routing Configuration

## Model Tiers

| Tier | Model | Role |
|---|---|---|
| **Local** | Ollama (model TBD) | Routine processing — free, fully on-device |
| **Cheap** | Claude Haiku | Secondary review — fast, low-cost |
| **Primary** | Claude Sonnet or Opus | Complex reasoning, sensitive analysis |

Ollama is not yet installed. When it is, the specific model will be documented here.

---

## Routing Rules

### System Monitoring

1. **Local (Ollama)** handles the initial read and analysis of monitoring output files.
   - If everything looks normal → summarize and done.
   - If something looks unusual but Ollama is uncertain → escalate to Haiku.

2. **Haiku** performs a second-pass review of flagged output.
   - If Haiku confirms no issue → summarize and done.
   - If Haiku confirms or suspects an anomaly → escalate to Primary with full context.

3. **Primary (Sonnet/Opus)** handles confirmed or suspected anomalies.
   - Performs deep analysis and produces a recommendation for Joe.

### Brokerage Analysis

Always routed directly to **Primary (Sonnet/Opus)**.
No routing through cheaper tiers — this work requires reliable reasoning.

### Work Monitoring (TBD)

Routing to be defined once the data delivery mechanism is established.
Expected: same three-tier pattern as system monitoring.

---

## Escalation Context

When escalating between tiers, pass:
- The original input (file name, timestamp, raw content)
- The lower tier's output and why it escalated
- Any relevant prior context from the same monitoring run

This ensures the higher tier has full context without re-reading the source data independently.

---

## Logging

All requests and responses at every tier are logged to `/mnt/openclaw/logs/`.
Log entries must include: timestamp, tier used, input summary, output summary, and escalation reason (if any).
