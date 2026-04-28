# Claude Code Handoff Context

This document summarizes the state of the OpenClaw setup project so that Claude Code
can continue where the previous session left off.

---

## What We Are Building

An OpenClaw AI agent gateway running on a Raspberry Pi 5, using Claude API as the
primary (most capable) LLM, with a prompt router to direct simpler queries to cheaper
or locally-running LLMs. The Pi has access to a dedicated NFS sharepoint which is the
sole filesystem resource OpenClaw is permitted to use.

---

## Hardware

- Raspberry Pi 5, 4GB RAM (an 8GB unit is available to swap in if memory pressure becomes an issue)
- 64GB SD card (no NVMe)
- Dedicated NFS sharepoint mounted on the Pi — path documented in `etc/fstab`
- 2GB swapfile configured on the SD card

---

## Security Constraints

These are non-negotiable and must be enforced at every layer (mount permissions,
OpenClaw config, and system prompt):

1. OpenClaw must not access any filesystem or network resource it has not been
   explicitly instructed to use. Its sole filesystem resource is the one designated
   NFS sharepoint.
2. Nothing should be sent to any LLM without the operator being able to control
   and audit it.
3. Outbound network access should be locked down to explicitly allowlisted endpoints
   (e.g. `api.anthropic.com`).
4. OpenClaw should run as a non-root user with permissions scoped accordingly.
5. All LLM requests and responses should be logged (to the NFS sharepoint).

---

## LLM Strategy

- **Primary/most capable:** Claude API (Anthropic)
- **Simpler/cheaper queries:** routed to a less expensive model (e.g. Claude Haiku)
  or a locally running LLM (e.g. via Ollama) — routing logic TBD
- A **prompt router** is to be configured to make these dispatch decisions

---

## Repo File Structure

| File | Purpose |
|---|---|
| `SETUP.md` | Step-by-step setup log, updated as we proceed |
| `CONTEXT.md` | This file — handoff context for Claude Code |
| `CRABBY.md` | Crabby system prompt, persona, and behavioral boundaries |
| `CRABBY_ROUTING.md` | Prompt routing logic and model dispatch configuration |
| `etc/` | System config files mirroring their paths on the Pi |

`CRABBY_ROUTING.md` has not been created yet — it is next.

---

## Where We Stopped

Setup is complete and the gateway is running. See SETUP.md for the full log.

Current state:
- Gateway running as a user-level systemd service on the Pi
- Telegram channel active, locked to Joe's user ID
- Claude Opus as primary model
- Haiku as cheap/secondary tier (local Ollama tier suspended — see SETUP.md Step 6)
- Runtime logs forwarded to `/mnt/openclaw/logs/gateway.log` via log forwarder service
- Ollama installed with `llama3.2:3b` and `llama3.2:3b-claw` pulled, but not usable via OpenClaw
  due to OpenClaw hardcoding `num_ctx=131072` in API calls (exceeds Pi memory)

Remaining tasks are tracked in SETUP.md under "Remaining Tasks".

---

## Working Style

Please continue this setup **one step at a time**. Do not list multiple steps ahead —
if something goes wrong at step 2, subsequent steps become noise. Wait for confirmation
before proceeding to the next step.