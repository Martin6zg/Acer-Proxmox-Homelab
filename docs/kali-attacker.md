# Kali Linux Attacker VM on Proxmox

Attacker VM for homelab penetration testing practice. Built for use against lab targets you own or have explicit permission to test.

## Overview

- **Hostname:** kali-attacker
- **OS:** Kali Linux (x86_64 Installer ISO)
- **Hypervisor:** Proxmox VE (QEMU/KVM)
- **Desktop:** Xfce (Kali default, lightweight)
- **Toolset:** top10 + default (includes nmap, Wireshark, Burp Suite, Metasploit)

## Recommended Specs

| Component | Setting |
| --- | --- |
| CPU | 4 cores, Type: host |
| Memory | 8192 MB (4096 MB minimum) |
| Disk | 80 GB, VirtIO SCSI single, SSD emulation + Discard + IO thread enabled |
| Network | VirtIO paravirtualized |
| BIOS | SeaBIOS |
| Machine | q35 |
| QEMU Guest Agent | Enabled |

## Prerequisites

1. Proxmox VE node up and running
2. Storage available for ISOs (`local`) and VMs (`local-lvm` or similar)
3. Kali Linux 64-bit Installer ISO from https://www.kali.org/get-kali/
   - Use **Installer Images**, not the VMware/VirtualBox prebuilt images
   - Select platform **x86_64**

## Step 1 - Upload ISO to Proxmox

1. In Proxmox, select your node > `local` storage > **ISO Images**
2. Click **Upload** and select the `kali-linux-*-installer-amd64.iso` file
   - Or use **Download from URL** and paste the direct Kali download link
3. Wait for upload to complete before creating the VM

## Step 2 - Create the VM

In Proxmox, click **Create VM** and use:

**General**
- Name: `kali-attacker`
- Notes: attacker VM for homelab

**OS**
- ISO image: select the Kali installer ISO
- Type: Linux
- Version: 6.x - 2.6 Kernel (or 7.x if shown)

**System**
- Machine: q35
- BIOS: SeaBIOS
- SCSI Controller: VirtIO SCSI single
- Check: QEMU Agent

**Disks**
- Storage: your VM storage
- Disk size: 80G
- Check: SSD emulation, Discard, IO thread

**CPU**
- Cores: 4
- Type: host

**Memory**
- Memory: 8192
- Leave Ballooning Device checked

**Network**
- Bridge: `vmbr0` (or your dedicated lab bridge like `vmbr1`)
- Model: VirtIO (paravirtualized)

Review and click **Finish**, then **Start** the VM and open **Console**.

## Step 3 - Install Kali Linux

1. Boot menu: Select **Graphical install**
2. Language, location, keyboard: set to your preferences
3. Hostname: `kali`
4. Domain name: leave blank, or use `home.arpa` / `homelab.local`
   - Do not use a public domain you do not own
5. Create user account and password when prompted
6. Partition disks:
   - Select **Guided - use entire disk**
   - Select your virtual disk (sda / vda)
   - Select **All files in one partition (recommended for new users)**
   - Select **Finish partitioning and write changes to disk**
   - When asked **Write the changes to disks?** select **Yes**
7. Install GRUB boot loader:
   - Install the GRUB boot loader to your primary drive? **Yes**
   - Device for boot loader installation: select `/dev/sda`
8. Software selection - leave defaults:
   - [x] Desktop environment
   - [x] ... Xfce (Kali's default desktop environment)
   - [ ] ... GNOME
   - [ ] ... KDE Plasma
   - [x] Collection of tools
   - [x] ... top10 -- the 10 most popular tools
   - [x] ... default -- recommended tools (available in the live system)
9. Click **Continue** and wait for install to complete
10. Reboot when prompted

After first boot, in Proxmox go to VM > **Hardware** > **CD/DVD Drive** and remove the Kali ISO or set to Do not use any media.

## Step 4 - Post-Install Setup

Open a terminal in Kali:

sudo apt update && sudo apt full-upgrade -y
sudo apt install qemu-guest-agent -y
sudo systemctl enable --now qemu-guest-agent


In Proxmox:
1. VM > **Options** > **QEMU Guest Agent** - confirm enabled
2. Verify guest agent shows IP address in VM **Summary** page

## Step 5 - Snapshot and Lab Networking

1. Take a clean snapshot:
   - VM > **Snapshots** > **Take Snapshot**
   - Name: `clean-install`
   - Description: fresh Kali install with guest agent

2. Homelab networking tip:
   - Place this VM and your vulnerable target VMs on the same isolated bridge
   - Example: create `vmbr1` with no physical port for isolated lab-only traffic
   - Keep lab traffic off your main LAN

## Responsible Use

Only scan, test, or exploit systems you own or have written permission to test. Keep attacker and target VMs on an isolated lab network.

## Troubleshooting

- **No ISO in dropdown:** ISO not finished uploading to `local` storage yet
- **No network in VM:** check bridge selection and VirtIO model
- **Guest agent no IP:** ensure `qemu-guest-agent` is installed and running, then reboot VM
