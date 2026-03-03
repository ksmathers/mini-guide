---
title: lxc-bluetti-monitor
description: 
published: true
date: 2026-01-18T00:00:00.000Z
tags: 
editor: markdown
dateCreated: 2026-01-18T00:00:00.000Z
---

# Bluetti Bluetooth Monitor in LXC Container (Mini Guide)

This guide covers setting up a Proxmox LXC container to monitor Bluetti power stations via Bluetooth using the hassio-bluetti-bt project.

## 📋 Overview

**What this does**: Monitor Bluetti power stations (AC300, AC500, EP600, etc.) over Bluetooth and collect metrics like battery charge, power input/output, and other sensor data.

**Key Features**:
- Direct Bluetooth adapter passthrough to container
- Python-based monitoring service
- Supports multiple Bluetti device models
- YAML-based device configuration

## 🔧 Part 1: Create LXC Container

### 1. Create Container in Proxmox

In the Proxmox web UI:

1. Click **Create CT**
2. Configure the container:
   ```
   CT ID: (choose an available ID)
   Hostname: bluetti-monitor
   Password: (set a secure password)
   ```

3. **Template**: Debian 12 or Ubuntu 22.04 (recommended)

4. **Root Disk**: 8GB minimum (sufficient for Python and dependencies)

5. **CPU**: 1 core minimum

6. **Memory**: 512MB minimum, 1GB recommended

7. **Network**: Bridge to appropriate VLAN

8. **Important Options**:
   ```
   ✓ Unprivileged container: NO (must be privileged for Bluetooth access)
   ✓ Start at boot: YES (optional, for automatic monitoring)
   ✓ Nesting: NO (not required)
   ```

9. Click **Finish** but don't start yet

## 🔌 Part 2: Bluetooth Adapter Passthrough

### 2. Identify Host Bluetooth Adapter

SSH into your Proxmox host:

```bash
# List USB devices
lsusb

# Identify Bluetooth adapter (look for Bluetooth)
# Example output:
# Bus 001 Device 003: ID 8087:0a2a Intel Corp. Bluetooth wireless interface
```

Alternative method using device path:

```bash
# Find Bluetooth device
ls -l /dev/serial/by-id/ | grep -i bluetooth
# or
hciconfig -a
# Note the hci number (usually hci0)
```

### 3. Configure Container for Bluetooth Access

Edit the container configuration file on the Proxmox host:

```bash
# Replace 100 with your container ID
nano /etc/pve/lxc/100.conf
```

Add these lines at the end:

```bash
# Bluetooth device passthrough
lxc.cgroup2.devices.allow: c 10:* rwm
lxc.cgroup2.devices.allow: c 226:* rwm
lxc.mount.entry: /dev/bus/usb dev/bus/usb none bind,optional,create=dir
lxc.mount.entry: /run/udev/data run/udev/data none bind,optional,create=dir

# For direct Bluetooth device access
lxc.cgroup2.devices.allow: c 189:* rwm
```

**Alternative method** - USB device passthrough (if above doesn't work):

Find the USB vendor and product ID from `lsusb`:
```bash
lsusb | grep -i bluetooth
# Example: Bus 001 Device 003: ID 8087:0a2a Intel Corp.
#                                    ^^^^ ^^^^
#                                  vendor:product
```

Add to container config:
```bash
# USB passthrough (replace vendor:product with your values)
lxc.cgroup2.devices.allow: a
usb0: host=8087:0a2a
```

### 4. Start the Container

```bash
pct start 100
```

## 📦 Part 3: Container Setup

### 5. Install System Dependencies

Enter the container:

```bash
pct enter 100
```

Update and install required packages:

```bash
apt update
apt upgrade -y
apt install -y \
    python3 \
    python3-pip \
    python3-venv \
    git \
    bluetooth \
    bluez \
    bluez-tools \
    libbluetooth-dev \
    pkg-config
```

### 6. Verify Bluetooth Access

Test Bluetooth functionality:

```bash
# Check if Bluetooth adapter is visible
hciconfig

# Should show output like:
# hci0:   Type: Primary  Bus: USB
#         BD Address: XX:XX:XX:XX:XX:XX  ACL MTU: 1021:8  SCO MTU: 64:1

# Enable the adapter if needed
hciconfig hci0 up

# Scan for nearby Bluetooth devices
bluetoothctl scan on
# Let it scan for 10-20 seconds, then:
# scan off
# quit
```

If you see your Bluetti device listed, Bluetooth passthrough is working correctly.

### 7. Bluetooth Pairing (Optional)

Some Bluetti devices may require pairing:

```bash
bluetoothctl
# Inside bluetoothctl:
scan on
# Wait to see your device
pair XX:XX:XX:XX:XX:XX
trust XX:XX:XX:XX:XX:XX
scan off
quit
```

**Note**: Most Bluetti devices don't require traditional Bluetooth pairing, but use encryption keys configured in `bluetti.yaml`.

## 🐍 Part 4: Install Python Application

### 8. Clone the Project

Create application directory and clone:

```bash
cd /opt
git clone https://github.com/ksmathers/hassio-bluetti-bt.git
cd hassio-bluetti-bt
```

### 9. Create Python Virtual Environment

```bash
# Create virtual environment
python3 -m venv venv

# Activate virtual environment
source venv/bin/activate

# Install Python dependencies
pip install --upgrade pip
pip install bleak
pip install -r requirements.txt
```

### 10. Configure Device

Edit the configuration file:

```bash
nano bluetti.yaml
```

Configure your Bluetti device:

```yaml
- address: XX:XX:XX:XX:XX:XX  # Your device's Bluetooth MAC address
  name: AP3002526010528096     # Device name/serial (optional)
  encryption: true              # Set to true if device requires encryption
  interval: 10                  # Polling interval in seconds
  sensors:
    - power_input               # Solar/AC input power
    - power_output              # AC output power
    - current_battery_charge    # Battery percentage
    # Add more sensors as needed
```

**Finding your device address**:
- Use `bluetoothctl scan on` to find the MAC address
- Bluetti devices usually appear as "BT-THxxxxxxxxx" or similar
- The address format is: `XX:XX:XX:XX:XX:XX`

**Common sensors** (device-dependent):
- `power_input` - Input power (solar/AC)
- `power_output` - Output power
- `current_battery_charge` - Battery %
- `ac_output_on` - AC output status
- `dc_output_on` - DC output status
- `total_battery_voltage` - Battery voltage
- `internal_ac_voltage` - AC voltage
- `internal_current_one` - Current measurement

### 11. Test the Monitor

Run the monitor manually to verify configuration:

```bash
# Make sure virtual environment is activated
source /opt/hassio-bluetti-bt/venv/bin/activate

# Run the monitor
cd /opt/hassio-bluetti-bt
python bluetti-monitor.py
```

You should see output showing connection and sensor readings:

```
Connecting to device XX:XX:XX:XX:XX:XX...
Connected successfully
Reading sensors...
  power_input: 250W
  power_output: 150W
  current_battery_charge: 85%
```

Press `Ctrl+C` to stop the test.

## 🚀 Part 5: Systemd Service (Optional)

### 12. Create Systemd Service

To run the monitor automatically as a service:

```bash
nano /etc/systemd/system/bluetti-monitor.service
```

Add the following content:

```ini
[Unit]
Description=Bluetti Bluetooth Monitor
After=network.target bluetooth.target
Wants=bluetooth.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/hassio-bluetti-bt
Environment="PATH=/opt/hassio-bluetti-bt/venv/bin"
ExecStart=/opt/hassio-bluetti-bt/venv/bin/python /opt/hassio-bluetti-bt/bluetti-monitor.py
Restart=on-failure
RestartSec=30
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

### 13. Enable and Start Service

```bash
# Reload systemd
systemctl daemon-reload

# Enable service to start at boot
systemctl enable bluetti-monitor.service

# Start the service
systemctl start bluetti-monitor.service

# Check status
systemctl status bluetti-monitor.service

# View logs
journalctl -u bluetti-monitor.service -f
```

## 🔍 Troubleshooting

### Bluetooth Device Not Found

```bash
# Restart Bluetooth service
systemctl restart bluetooth

# Verify adapter is up
hciconfig hci0 up

# Check if device is in range
bluetoothctl scan on
```

### Permission Denied Errors

```bash
# Ensure container is privileged (not unprivileged)
# Check on Proxmox host:
grep "unprivileged" /etc/pve/lxc/100.conf
# Should show: unprivileged: 0
```

### Connection Timeout

- Verify the MAC address in `bluetti.yaml` is correct
- Check if `encryption: true` is set correctly for your device
- Ensure the Bluetti device is powered on and within range
- Try increasing the `interval` value to reduce polling frequency

### Python Package Issues

```bash
# Reinstall in clean environment
cd /opt/hassio-bluetti-bt
rm -rf venv
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install bleak
pip install -r requirements.txt
```

### USB Device Passthrough Not Working

If Bluetooth adapter isn't visible in container:

1. On Proxmox host, find exact USB path:
   ```bash
   ls -la /dev/bus/usb/001/  # Check all /dev/bus/usb/00*/
   ```

2. Add specific device mapping to container config:
   ```bash
   nano /etc/pve/lxc/100.conf
   ```
   
   ```bash
   lxc.mount.entry: /dev/bus/usb/001/003 dev/bus/usb/001/003 none bind,optional,create=file
   ```

3. Restart container:
   ```bash
   pct stop 100
   pct start 100
   ```

## 📊 Monitoring Output

The monitor will continuously poll your Bluetti device and output readings. You can:

1. **View live logs**:
   ```bash
   journalctl -u bluetti-monitor.service -f
   ```

2. **Export data**: Modify `bluetti-monitor.py` to log to file or send to monitoring system

3. **Integration**: The project includes Home Assistant integration components in `custom_components/`

## 🔗 Additional Resources

- **Project Repository**: https://github.com/ksmathers/hassio-bluetti-bt
- **Original Project**: https://github.com/Patrick762/hassio-bluetti-bt
- **Supported Devices**: See README.md for complete list of supported Bluetti models
- **Device Probe Tool**: Use `probe-device.py --scan` to identify device protocol

## ✅ Summary

You now have a dedicated LXC container running a Bluetooth monitor for your Bluetti power station with:

- ✅ Bluetooth adapter passthrough to container
- ✅ Python environment with Bleak and dependencies
- ✅ hassio-bluetti-bt monitoring application
- ✅ YAML-based device configuration
- ✅ Optional systemd service for automatic monitoring
- ✅ Device pairing and search capabilities

The monitor will continuously collect metrics from your Bluetti device, which can be used for logging, alerting, or integration with home automation systems.
