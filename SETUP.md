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
| Firewall | Installs `nftables`; inbound: SSH only; outbound: DNS, NTP, NFS to `pi-nas`, HTTPS to `api.anthropic.com` only |
| fstab | Adds `noexec,nosuid` to the NFS mount entry |

After the script completes, verify with:

```bash
id openclaw
nft list ruleset
ls -ld /mnt/openclaw
```

<!-- Results will be recorded here after running on the Pi. -->
