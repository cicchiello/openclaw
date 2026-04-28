# Crabby — Prompt Routing Configuration

## Model Tiers

| Tier | Model | Role |
|---|---|---|
| ~~**Local**~~ | ~~Ollama~~ | ~~Routine processing — free, fully on-device~~ |
| **Cheap** | Claude Haiku | First-pass review — fast, low-cost |
| **Primary** | Claude Sonnet or Opus | Complex reasoning, sensitive analysis |

**Local tier is suspended.** Ollama is installed and `llama3.2:3b` runs successfully via direct
`ollama run` (~6–27s depending on context size), but OpenClaw hardcodes `num_ctx=131072` in its
Ollama API calls, causing the Pi to run out of memory. All known config-level workarounds were
exhausted. See SETUP.md Step 6 for full details.

The local tier may be revisited using **llama.cpp server** instead of Ollama — it enforces
`--ctx-size` at the socket level and cannot be overridden by the API caller.

---

## Routing Rules

### System Monitoring

1. **Haiku** handles the initial read and analysis of monitoring output files.
   - If everything looks normal → summarize and done.
   - If something looks unusual but Haiku is uncertain → escalate to Primary.

2. **Primary (Sonnet/Opus)** handles confirmed or suspected anomalies.
   - Performs deep analysis and produces a recommendation for Joe.

*(Three-tier routing with a local first pass is the goal — restore once local model tier is working.)*

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
