# Thermal Shutdown Protection for Proxmox

Automatically shuts down the Proxmox node if CPU temp hits 85C. For Acer Aspire M5-583P (i5-4200U, TJMax 100C).

## 1. Install sensors

```bash
apt update
apt install lm-sensors -y
sensors-detect --auto
sensors
```

## 2. Create script

File: `/usr/local/bin/temp-shutdown.sh`

```bash
#!/bin/bash
THRESHOLD=85
LOGFILE="/var/log/tempmon.log"
RAW=$(cat /sys/class/thermal/thermal_zone0/temp)
TEMP_C=$((RAW/1000))

if [ "$TEMP_C" -ge "$THRESHOLD" ]; then
  MSG="$(date): CRITICAL temp ${TEMP_C}C >= ${THRESHOLD}C, shutting down"
  echo "$MSG" >> "$LOGFILE"
  logger -t tempmon "$MSG"
  shutdown -h now "Critical temperature ${TEMP_C}C reached"
fi
```

```bash
chmod +x /usr/local/bin/temp-shutdown.sh
```

## 3. Run every minute

```bash
crontab -e
```

Add:

```
* * * * * /usr/local/bin/temp-shutdown.sh
```

## Notes

- 85C recommended, 90C max. Do not use 95C+.
- `shutdown -h now` triggers pve-guests.service to stop VMs cleanly.
- Check logs with: `cat /var/log/tempmon.log` and `journalctl -t tempmon`
