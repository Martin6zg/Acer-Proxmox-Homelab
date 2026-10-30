# Acer Aspire Proxmox Homelab - Build and Operations Guide

**Version:** 1.0 | **Date:** October 29, 2026 | **Author:** Martin Godinez  
**Status:** In Progress - Windows 11 VM build ongoing

> Real IPs, device names, and credentials replaced with [PLACEHOLDERS]. Secrets stored in password manager.

## Table of Contents
1. [Project Overview](#1-project-overview)
2. [Goals and Use Cases](#2-goals-and-use-cases)
3. [Hardware Inventory](#3-hardware-inventory)
4. [Software Stack](#4-software-stack)
5. [Network Architecture](#5-network-architecture)
6. [Build Log](#6-build-log)
7. [Configuration Details](#7-configuration-details)
8. [Services and Workloads](#8-services-and-workloads)
9. [Security Baseline](#9-security-baseline)
10. [Maintenance and Monitoring](#10-maintenance-and-monitoring)
11. [Backup and Recovery](#11-backup-and-recovery)
12. [Troubleshooting Log](#12-troubleshooting-log)
13. [Lessons Learned and Roadmap](#13-lessons-learned-and-roadmap)
- [Appendix A: Quick Reference](#appendix-a-quick-reference)
- [Appendix B: Revision History](#appendix-b-revision-history)

## 1. Project Overview

This document describes the design, build, and operational configuration of a low-cost, low-power home lab built from a repurposed Acer Aspire M5-583P-6423 laptop. The system was upgraded with additional RAM and a solid-state drive, then provisioned as a Proxmox Virtual Environment (VE) hypervisor host.

The build emphasizes reuse of existing hardware, secure remote access without port forwarding, and reproducible documentation.

## 2. Goals and Use Cases

- Create an affordable virtualization platform for IT support and cybersecurity practice
- Run Windows 11 VM and Linux (Ubuntu Server) VM for cross-platform administration
- Practice core ops: incident troubleshooting, updates, access management, backup
- Enable secure remote admin via Tailscale, no inbound firewall ports
- Operate headless (lid closed) 24/7

## 3. Hardware Inventory

| Component | Specification / Notes |
|-----------|-----------------------|
| Model | Acer Aspire M5-583P-6423, 15.6" touchscreen |
| CPU | Intel Core i5-4200U (Haswell), 2 cores / 4 threads, 1.6 GHz / 2.6 GHz Turbo, 15W |
| Virtualization | Intel VT-x with EPT, VT-d, AES-NI, AVX2 |
| Memory Stock | 6 GB DDR3L |
| Memory Upgraded | 12 GB total (4 GB soldered + 8 GB DDR3L / PC3L 1.35V) |
| Storage Stock | 500 GB HDD (removed) |
| Storage Upgraded | 512 GB SATA SSD |
| Network | Wired Ethernet for Proxmox bridge |

**Suitability:** 2 cores suits light services and a few small VMs. Favor LXC/Docker for light services, full VMs only when needing own kernel.

### Upgrade Procedure
1. Shutdown, unplug, open bottom panel per Acer manual. Disconnect battery if instructed.
2. Discharge static, photograph layout.
3. Install RAM at ~30° angle until clicks. Use DDR3L / PC3L 1.35V.
4. Mount SSD in HDD bracket to SATA.
5. Reconnect, close, press F2 to confirm BIOS sees 12GB + 512GB.

Verified with: Task Manager > Performance, CPU-Z, Disk Management, CrystalDiskInfo, memtest86+ (1 pass).

## 4. Software Stack

| Layer | Product | Purpose |
|-------|---------|---------|
| Hypervisor | Proxmox VE (No-Subscription) | VMs, LXC, snapshots, backups |
| Remote Access | Tailscale | Private encrypted mesh VPN |
| Guest 1 (planned) | Windows 11 x64 | Windows admin practice |
| Guest 2 (planned) | Ubuntu Server | Linux admin practice |
| Drivers | VirtIO ISO | Paravirtualized drivers |

Proxmox chosen over Ubuntu + KVM/Cockpit and Docker-only for integrated web UI and backup workflow.

## 5. Network Architecture

Single-host flat LAN + encrypted overlay. No port forwarding.

| Item | Value |
|------|-------|
| Hostname | [node name] |
| Router | [ROUTER-IP] |
| Proxmox LAN | [SERVER-IP] static, wired |
| Web UI LAN | https://[SERVER-IP]:8006 |
| Tailscale IP | [TAILSCALE-IP] |
| Web UI Remote | https://[TAILSCALE-IP]:8006 |
| DNS | Router IP + 1.1.1.1 |
| Admin devices | Laptop + phone on Tailscale |

> Reserve [SERVER-IP] in router DHCP reservations.

## 6. Build Log

Build window: Oct 28-29, 2026

### 6.1 Proxmox VE Installation
- Installed to 512GB SSD (wipes disk).
- Console login failed with personal username: `Login incorrect`. **Fix:** Use `root` + install password.

### 6.2 Network Correction
Symptom: Web UI unreachable. `ip a` showed wrong subnet.
```bash
nano /etc/network/interfaces  # set address [SERVER-IP]/24 and gateway [ROUTER-IP]
nano /etc/hosts                # replace old IP
reboot
```
Later `NO-CARRIER` fixed by reseating cable. Result: `https://[SERVER-IP]:8006` reachable. Login: root, realm Linux PAM.

### 6.3 Repository Configuration
`Update package database` failed - enterprise repo needs subscription.
Fix: Node > Updates > Repositories > disable `pve-enterprise`, add No-Subscription, Refresh > Upgrade.

### 6.4 Headless Operation
```bash
nano /etc/systemd/logind.conf
# Set:
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
reboot
```
Pitfalls: `logind.config` is wrong file, use `logind.conf`. Verify with `grep -i HandleLidSwitch /etc/systemd/logind.conf` - no `#` should remain.

### 6.5 Tailscale Remote Access
Policy: never forward port 8006.
```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```
Installed on phone/laptop, disabled key expiry on server. Tested on cellular: `https://[TAILSCALE-IP]:8006` works.

## 7. Configuration Details

- Login: root / Linux PAM, password in manager. Watch phone auto-capitalization.
- Resource plan (12GB): 1-1.5GB host, 4GB Windows, 2GB Ubuntu, leave headroom. vCPUs ~4 total. No overcommit. Thin provision.
- Storage: single SSD, keep 10-20% free, check SMART monthly.
- Power: battery as mini-UPS, monitor health, remove if swelling. Vents clear, hard surface.

## 8. Services and Workloads

### Windows 11 VM (win11) - Planned
| Tab | Setting |
|-----|---------|
| General | Name: win11 |
| OS | Windows 11 ISO, Type Microsoft Windows |
| System | q35, OVMF (UEFI), EFI disk, VirtIO SCSI single, TPM |
| Disks | SCSI, 64 GB |
| CPU | 2 cores, type host |
| Memory | 4096 MB |
| Network | VirtIO |

- ISO from microsoft.com, upload to local > ISO Images
- If no drives: Load driver > `vioscsi\w11\amd64` from VirtIO CD
- Post-install: `virtio-win-gt-x64.exe`
- Bypass internet: `Shift+F10`, `OOBE\BYPASSNRO`

Future candidates: Pi-hole, WireGuard, Uptime Kuma, Home Assistant.

## 9. Security Baseline

- LAN-only direct, remote via Tailscale only
- Self-signed cert warning expected, traffic encrypted
- Done: strong root password, no port forward
- TODO: 2FA (Datacenter > Permissions > Two Factor), non-root admin, SSH key-only, Proxmox firewall, auto-updates

## 10. Maintenance and Monitoring

- Regular No-Subscription updates
- Check Node > Summary, SMART monthly, `sensors` for temps
- Reserve IP in router
- Re-test lid + Tailscale after reboot

## 11. Backup and Recovery

Single SSD = no redundancy. Backups must go to external drive/NAS via Datacenter > Backup. Test restore still required.

## 12. Troubleshooting Log

| Date | Issue | Root Cause | Resolution |
|------|-------|------------|------------|
| 2026-10-28 | Console login incorrect | Needs root, not username | Logged in as root |
| 2026-10-28 | Web UI unreachable | IP range mismatch | Set [SERVER-IP]/24, gateway [ROUTER-IP] |
| 2026-10-28 | NO-CARRIER | Loose cable | Reseated |
| 2026-10-29 | apt update fail | Enterprise repo | Enabled No-Subscription |
| 2026-10-29 | Lid still suspends | Typo config, # left | Fixed logind.conf |
| 2026-10-29 | curl resolve https fail | Typo https// | Fixed URL |
| 2026-10-29 | curl resolve tailscale.com fail | Stale DNS | Set DNS to router + 1.1.1.1 |
| 2026-10-29 | tailscale not found | Incomplete install | Re-ran script |
| 2026-10-29 | apt Not live until | Clock behind | `timedatectl set-ntp true` |

## 13. Lessons Learned and Roadmap

**Lessons:**
- Check L3 first: IP, DNS, clock cause most failures
- Filenames matter: `logind.conf` vs `logind.config`
- Plan thermals/power for laptop labs

**Roadmap:**
- [ ] Finish Windows 11 VM
- [ ] Create Ubuntu Server VM
- [ ] Reserve IP, enable 2FA
- [ ] Validate lid-closed + Tailscale after reboot
- [ ] Configure backups, test restore
- [ ] Baseline power/temps/disk speed

## Appendix A: Quick Reference

| Command | Purpose |
|---------|---------|
| `ip a` | Show IPs |
| `ping -c 3 [ROUTER-IP]` | Test router |
| `date` | Check clock |
| `timedatectl set-ntp true` | Enable NTP |
| `apt update` | Refresh packages |
| `tailscale status` | Tailscale state |
| `passwd` | Change password |
| `reboot` | Restart |

## Appendix B: Revision History

| Version | Date | Changes |
|---------|------|---------|
| 0.1 | 2026-10-28 | Hardware upgrades, install, network fix |
| 0.9 | 2026-10-29 | Headless, Tailscale, troubleshooting |
| 1.0 | 2026-10-29 | Formal documentation |

— End —
