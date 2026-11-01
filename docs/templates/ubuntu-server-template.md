# Ubuntu Server VM Template for Proxmox

Base Linux VM for Proxmox VE. Built as a reusable template for future lab VMs.
Covers command line, users/permissions, SSH, services, and logs.

## Specs

| Setting | Value |
| --- | --- |
| Proxmox Node | Marty |
| Proxmox Version | 9.2.2 |
| VM ID | 9000 |
| VM Name | ubuntu-server-template |
| OS | Ubuntu Server 26.04 LTS |
| ISO | ubuntu-26.04.1-live-server-amd64.iso |
| CPU | 2 cores |
| RAM | 4096 MB |
| Disk | 32 GB, VirtIO Block on local-lvm, Discard on |
| Network | vmbr0, VirtIO paravirtualized |
| BIOS | OVMF (UEFI), Machine q35 |
| EFI Disk | Yes, on local-lvm |

## 1. Upload ISO

1. In Proxmox, select node **Marty** > **Storage** > **local** > **ISO Images**
2. Click **Upload** and select your Ubuntu Server ISO
3. Wait for upload to complete

## 2. Create VM

Click **Create VM** and use:

**General**
- VM ID: `9000`
- Name: `ubuntu-server-template`

**OS**
- Use CD/DVD disc image file: select Ubuntu ISO on `local`
- Type: Linux, Version: 6.x - 2.6 Kernel

**System**
- Machine: `q35`
- BIOS: `OVMF (UEFI)`
- Check Add EFI Disk, Storage: `local-lvm`
- SCSI Controller: `VirtIO SCSI`
- Check Qemu Agent

**Disks**
- Bus/Device: VirtIO Block
- Storage: `local-lvm`
- Size: `32`
- Check Discard

**CPU**
- Sockets: 1, Cores: 2

**Memory**
- 4096 MB

**Network**
- Bridge: `vmbr0`
- Model: `VirtIO (paravirtualized)`

Check **Start after created**, then Finish.

After creation:
- Go to VM 9000 > **Options** > **QEMU Guest Agent** = Yes

## 3. Install Ubuntu Server

Open **Console** and install:

- Language: English
- Network: DHCP (note the IP, e.g. 192.168.0.x)
- Storage: Use an entire disk, Set up as LVM group. Do not encrypt.
- Profile: set hostname, username, and password
- SSH: Check **Install OpenSSH server**
- Server snaps: none

Reboot when done. Detach ISO: VM > Hardware > CD/DVD Drive > Do not use any media.

## 4. Post-install Setup

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y qemu-guest-agent htop curl vim git net-tools
sudo systemctl start qemu-guest-agent
```

Verify SSH:
```bash
sudo systemctl status ssh
ip a | grep 192.168
```

Optional base user:
```bash
sudo adduser practice
sudo usermod -aG sudo practice
```

## 5. Prepare for Template

Clean unique IDs so clones do not collide:

```bash
sudo apt autoremove -y
sudo apt clean
sudo truncate -s 0 /etc/machine-id
sudo rm /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id
history -c
sudo poweroff
```

## 6. Convert to Template

1. In Proxmox, ensure 9000 is **Stopped**
2. VM 9000 > Hardware > CD/DVD Drive = **Do not use any media**
3. Right-click 9000 > **Convert to template** > Yes

## 7. Clone from Template

1. Right-click template > **Clone**
2. New ID: e.g. `101`, Name: e.g. `lab-01`
3. Mode: **Full Clone**
4. Start the clone

Update hostname inside clone if needed:
```bash
sudo hostnamectl set-hostname lab-01
```

## Notes

- If `qemu-guest-agent` fails to start before first reboot, ensure QEMU Guest Agent is enabled in Proxmox Options and reboot the VM.
- Take a Proxmox Snapshot of clones before experimenting for easy rollback.
