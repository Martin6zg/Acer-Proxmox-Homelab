# Ubuntu Server VM in Proxmox

Build a reusable Ubuntu Server base image in Proxmox VE.
When finished, you can clone this image in seconds for future labs.

**What you will build:**
- Example VM ID `9000` named `ubuntu-base`
- Ubuntu Server 26.04 LTS with OpenSSH
- QEMU Guest Agent enabled
- Cleaned and converted to a Proxmox template

---

## Before You Start

You need:
- Access to Proxmox web UI (https://[PROXMOX-IP]:8006)
- Ubuntu Server ISO downloaded to your computer

---

## Part 1: Upload the Ubuntu ISO

1. In Proxmox, click your node name (e.g. `pve-node01`) in the left tree under Server View.
2. Click **Storage** in the middle panel, then select **local**.
3. Click **ISO Images** at the top.
4. Click **Upload**.
5. Select your Ubuntu Server ISO file (e.g. `ubuntu-26.04-live-server-amd64.iso`) and click **Upload**.
6. Wait until it appears in the ISO list. Do not continue until you see it.

## Part 2: Create the VM

1. Click **Create VM** in the top-right corner.
2. **General tab:**
   - VM ID: Type `9000`
   - Name: Type `ubuntu-base`
   - Click **Next**.
3. **OS tab:**
   - Select **Use CD/DVD disc image file (iso)**.
   - Storage: Select `local`.
   - ISO image: Select your Ubuntu ISO.
   - Type: `Linux`, Version: `6.x - 2.6 Kernel`.
   - Click **Next**.
4. **System tab:**
   - Machine: Select `q35`.
   - BIOS: Select `OVMF (UEFI)`.
   - Check **Add EFI Disk**.
   - EFI Storage: Select `local-lvm`.
   - SCSI Controller: Select `VirtIO SCSI`.
   - Check **Qemu Agent**.
   - Click **Next**.
5. **Disks tab:**
   - Leave Bus/Device as VirtIO Block.
   - Storage: `local-lvm`.
   - Disk size: Type `32`.
   - Check **Discard**.
   - Click **Next**.
6. **CPU tab:**
   - Sockets: `1`.
   - Cores: `2`.
   - Click **Next**.
7. **Memory tab:**
   - Memory: Type `4096`.
   - Click **Next**.
8. **Network tab:**
   - Bridge: Select `vmbr0`.
   - Model: Select `VirtIO (paravirtualized)`.
   - Click **Next**.
9. **Confirm tab:**
   - Review your settings.
   - Check **Start after created**.
   - Click **Finish**.

10. After it is created, select the VM in the left tree.
11. Go to **Options** > **QEMU Guest Agent**. Make sure it says **Yes**. If it says No, double-click it and enable it.

## Part 3: Install Ubuntu Server

1. Select the VM and click **Console**.
2. When the Ubuntu menu appears, select **Try or Install Ubuntu Server** and press Enter.
3. Select Language: **English**. Press Enter.
4. Select Keyboard: leave default. Select **Done** and press Enter.
5. Select Installation type: **Ubuntu Server**. Select **Done**.
6. Network: Leave DHCP enabled. Confirm you have an IP address (e.g. 192.168.1.50). Select **Done**.
7. Proxy: Leave blank. Select **Done**.
8. Mirror: Leave default. Select **Done**.
9. Guided storage:
   - Select **Use an entire disk**.
   - Check **Set up this disk as an LVM group**.
   - Do NOT check Encrypt.
   - Select **Done**.
10. Storage summary: Select **Done**, then select **Continue** to confirm.
11. Profile:
    - Your name: Example User
    - Your server's name: `ubuntu-base`
    - Pick a username: e.g. `labuser` (lowercase)
    - Choose a password and confirm it.
    - Select **Done**.
12. Ubuntu Pro: Select **Continue** / Skip.
13. SSH:
    - Check **Install OpenSSH server** (press Space to check).
    - Leave **Allow password authentication over SSH** checked.
    - Select **Done**.
14. Server snaps: Leave all unchecked. Select **Done**.
15. Wait for install to finish, then select **Reboot Now**.
16. When prompted, go back to Proxmox: VM > **Hardware** > **CD/DVD Drive** > **Edit** > **Do not use any media** > OK.

## Part 4: First Boot Configuration

1. In the Proxmox Console, log in with the username and password you created.
2. Update the system. Type your password when asked:
```bash
sudo apt update && sudo apt upgrade -y
```
3. Install required tools:
```bash
sudo apt install -y qemu-guest-agent htop curl vim git net-tools
```
4. Start the guest agent:
```bash
sudo systemctl start qemu-guest-agent
```
5. Verify SSH is running:
```bash
sudo systemctl status ssh
ip a | grep 192.168
```
6. After about 30 seconds, Proxmox should show the VM IP under Summary. This means the guest agent is working.

## Part 5: Prepare for Template

Run these to clean unique IDs so clones do not conflict:

```bash
sudo apt autoremove -y
sudo apt clean
sudo truncate -s 0 /etc/machine-id
sudo rm /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id
history -c
sudo poweroff
```

The VM will shut down. This is expected.

## Part 6: Convert to Template

1. In Proxmox, confirm the VM shows as **Stopped**.
2. Select the VM > **Hardware** > **CD/DVD Drive**. Ensure it is set to **Do not use any media**.
3. Right-click the VM in the left tree.
4. Click **Convert to template**.
5. Click **Yes** to confirm.

The icon will change. You cannot start a template directly. This is normal.

## Part 7: How to Use Your Template

1. Right-click the template > **Clone**.
2. VM ID: Type e.g. `101`.
3. Name: Type e.g. `test-vm01`.
4. Mode: Select **Full Clone**.
5. Click **Clone**.
6. Start the new VM.

Optional: Change hostname inside the clone:
```bash
sudo hostnamectl set-hostname test-vm01
```

Tip: In Proxmox, select your clone > **Snapshots** > **Take Snapshot** before experimenting. You can roll back in one click if you break something.
