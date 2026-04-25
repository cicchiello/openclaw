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

<!-- Steps will be added here as setup proceeds. -->
