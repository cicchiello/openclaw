# OpenClaw Setup

This repo documents the setup of an OpenClaw AI agent gateway running on a Raspberry Pi 5,
with access to a dedicated NFS sharepoint. All system configuration files are captured under
`etc/`, mirroring their paths on the system.

---

## Prerequisites

### OS Baseline

OS Version: Debian GNU/Linux 13 (trixie) (13.4, to be precise)

### Networking

Network configuration is managed via NetworkManager with dnsmasq. Configuration files are
captured in the repo:

- `etc/NetworkManager/conf.d/dnsmasq.conf` — NetworkManager dnsmasq configuration
- `etc/NetworkManager/dnsmasq.d/custom-hosts.conf` — Custom host entries

### NFS

There is a large filesystem being made available to OpenClaw.  It's accessible via an NFS share, rooted
at the folder that is reserved for OpenClaw work.

The NFS server is "pi-nas", with access to pi-nas defined between these 2 NetworkManager config files:
   <repo>/etc/NetworkManager/conf.d/dnsmasq.conf
   <repo>/etc/NetworkManager/dnsmasq.d/custome-hosts.conf

And the filesystem is defined in /etc/fstab per the example here:
   <repo>/etc/fstab
   
---

## OpenClaw Installation

### Step 1: System Hardening

Script: `scripts/01-harden.sh`

Run on the Pi as root:

```bash
sudo bash scripts/01-harden.sh
```

What it does:

| Area | Action |
|---|---|
| Safety net | Schedules a reboot in 5 minutes at the start; cancelled at the end if all steps succeed — ensures SSH access is restored if the firewall misconfigures |
| User/group | Creates system user `openclaw` (no shell, no home dir) |
| NFS mount | Asserts `/mnt/openclaw` is mounted, sets owner to `openclaw:openclaw`, mode `750` |
| Firewall | Installs `nftables`; inbound: SSH only; outbound: DNS, NTP, NFS to `pi-nas`, HTTPS (port 443, any destination — IP-pinning is impractical for CDN-backed services) |
| fstab | Adds `noexec,nosuid` to the NFS mount entry |

After the script completes, verify with:

```bash
id openclaw
nft list ruleset
ls -ld /mnt/openclaw
```

Results:

- User `openclaw` created (uid=999, gid=985, no shell, no home dir)
- `/mnt/openclaw` owned by `openclaw:openclaw`, mode `750`, mounted with `noexec,nosuid`
- `nftables` active — inbound SSH and LAN ICMP only; outbound DNS, NTP, NFS to `pi-nas`, HTTPS (port 443, any destination), LAN ICMP
- `/etc/fstab` corrected (swapfile entry was concatenated onto NFS line; split into two lines), NFS entry updated with `vers=4,noexec,nosuid`, and path corrected to `pi-nas:/openclaw` (NFSv4 path relative to export root)

### Step 2: Crabby System Prompt

Created `CRABBY.md` — defines Crabby's identity, filesystem boundaries, network constraints,
data handling rules, and current task domains (system monitoring, brokerage analysis).

### Step 3: Prompt Routing

Created `CRABBY_ROUTING.md` — defines three-tier model routing:
- **Local (Ollama):** routine monitoring (Ollama not yet installed)
- **Cheap (Claude Haiku):** secondary review of flagged monitoring output
- **Primary (Claude Sonnet/Opus):** anomalies, all brokerage analysis

### Step 4: OpenClaw Onboard

Ran `openclaw onboard` (Manual mode). Configured:
- Primary model: `anthropic/claude-opus-4-7`
- Gateway port: `18789`, bound to `0.0.0.0`
- Gateway auth: password
- Chat channel: Telegram (Bot API), dmPolicy set to `allowlist` with Joe's user ID
- Plugin: `@openclaw/ollama-provider` installed (discovery disabled until Ollama is set up)
- Hook: `session-memory` enabled
- Crabby initialized via TUI with `CRABBY.md` copied to `/mnt/openclaw/CRABBY.md`

Gateway runs as a user-level systemd service. To restart:

```bash
systemctl --user restart openclaw-gateway.service
```

### Step 5: Post-Onboard Configuration

#### Telegram allowlist
Locked down bot to Joe's Telegram user ID only:
```bash
openclaw config set channels.telegram.dmPolicy "allowlist"
openclaw config set channels.telegram.allowFrom '["<JOE_TELEGRAM_ID>"]'
systemctl --user restart openclaw-gateway.service
```

#### Control UI firewall rule
Added inbound port 18789 to nftables (also added to `scripts/01-harden.sh`):
```bash
sudo nft add rule inet filter input tcp dport 18789 accept
sudo nft list ruleset | sudo tee /etc/nftables.conf
```

Access via SSH tunnel for secure context (required for device identity features):
```bash
ssh -N -L 18789:127.0.0.1:18789 joe@10.0.0.52
```
Then open `http://localhost:18789` in browser.

#### Ollama installation
```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama pull llama3.2:3b
```

Moved model storage to NFS:
```bash
sudo systemctl stop ollama
sudo mv /usr/share/ollama/.ollama /mnt/openclaw/.ollama
sudo mkdir -p /etc/systemd/system/ollama.service.d
sudo tee /etc/systemd/system/ollama.service.d/override.conf <<EOF
[Service]
Environment="OLLAMA_MODELS=/mnt/openclaw/.ollama/models"
EOF
sudo systemctl daemon-reload
sudo systemctl start ollama
```

Enabled Ollama plugin in OpenClaw:
```bash
openclaw config set plugins.entries.ollama.enabled true
openclaw config set plugins.entries.ollama.config.discovery.enabled true
```

#### Pi startup optimizations
```bash
mkdir -p /var/tmp/openclaw-compile-cache
mkdir -p ~/.config/systemd/user/openclaw-gateway.service.d
cat > ~/.config/systemd/user/openclaw-gateway.service.d/override.conf <<'EOF'
[Service]
ExecStartPre=/bin/bash -c 'for i in $(seq 1 30); do mountpoint -q /mnt/openclaw && exit 0; sleep 2; done; exit 1'
Environment="NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache"
Environment="OPENCLAW_NO_RESPAWN=1"
EOF
systemctl --user daemon-reload
```

Note: `After=remote-fs.target` in a user unit is silently ignored — the user manager runs
as a separate systemd instance and cannot see system-level targets. The `ExecStartPre`
poll loop is the correct fix. It checks for the NFS mount every 2 seconds for up to 60
seconds before allowing the gateway to start.

Note: `OPENCLAW_STATE_DIR` was initially set to `/mnt/openclaw/.openclaw-state` to reduce
SD card write wear. This was reverted — the exec tool has `~/.openclaw` partially
hardcoded and `OPENCLAW_STATE_DIR` does not fully redirect it, breaking the overnight
monitoring job. State stays at `~/.openclaw`; logs go to `/mnt/openclaw/logs/` via the
log forwarder service (see below).

#### Logging

`logging.file` in OpenClaw config appears broken/unimplemented in v2026.4.23 — setting it
has no effect. Runtime logs are instead captured via a forwarder service that pipes
`openclaw logs --follow` to the NFS sharepoint.

Service file is tracked at `etc/systemd/user/openclaw-log-forwarder.service`. Deploy:

```bash
cp ~/openclaw/etc/systemd/user/openclaw-log-forwarder.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now openclaw-log-forwarder.service
```

Logs are written to `/mnt/openclaw/logs/gateway.log` in JSON format.

Note: `BindsTo=openclaw-gateway.service` ensures the forwarder stops and restarts with
the gateway.

Note: LLM conversation content (what is sent to/from models) is **not** in the runtime
log at `info` level. Main session transcripts are stored as JSONL files under
`~/.openclaw/agents/main/sessions/`. Embedded/subagent dispatches (e.g. Ollama) are not
persisted by default — `diagnostics.cacheTrace` is the mechanism to capture those
payloads but has not been configured yet.

### Remaining Tasks

- [ ] Ollama — verify it now appears in `openclaw models list` (was timing out on 4GB Pi; 8GB swap + boot ordering fix should resolve)
- [ ] HTTPS for Control UI (deferred — SSH tunnel works for now)
- [ ] SOUL.md review (Crabby rewrote it during init — verify it matches CRABBY.md)
- [ ] Work monitoring data path (TBD — how to get work data to Crabby without email access)

