---
author: "Felle"
title: "How to create a custom VPN toggle script for Waybar"
date: "2026-01-18"
description: "Tutorial showcasing how to build a custom VPN toggle script with random server selection for Waybar using ProtonVPN and OpenVPN."
tags: [
    "GNU/Linux",
    "Tutorials",
    "VPN",
]
categories: [
    "GNU/Linux",
    "Tutorials",
]
series: ["Tutorials"]
---

This tutorial demonstrates how to build a custom VPN toggle script for Waybar that integrates with ProtonVPN, provides random server selection, and sends desktop notifications.
<!--more-->

## Overview

Managing VPN connections on Linux through terminal commands gets tedious fast. This guide walks through building a Waybar module that:

- Shows current VPN status and connected country
- Toggles VPN with a single click
- Randomly selects from available servers
- Sends desktop notifications for connection events
- Handles multiple ProtonVPN configs automatically

The final result is about 100 lines of bash across two scripts that integrate seamlessly with your status bar.

## Prerequisites

Before starting, ensure you have:

- Basic bash scripting knowledge
- Linux system with systemd
- ProtonVPN account (free or paid)
- Waybar installed and configured
- Root/sudo access for system configuration

## Environment Setup

### Required Packages

Install OpenVPN and notification support:

```bash
# Arch Linux
sudo pacman -S openvpn libnotify

# Ubuntu/Debian  
sudo apt install openvpn libnotify-bin

# Fedora
sudo dnf install openvpn libnotify
```

### ProtonVPN Configuration Files

Download OpenVPN configs from [account.protonvpn.com](https://account.protonvpn.com/downloads) under Downloads → OpenVPN configuration files. Select the servers you want to use.

Install the downloaded configs:

```bash
# Move configs to OpenVPN directory
sudo mv ~/Downloads/*.ovpn /etc/openvpn/client/

# Rename .ovpn to .conf (systemd requirement)
cd /etc/openvpn/client/
for file in *.ovpn; do
    sudo mv "$file" "${file%.ovpn}.conf"
done
```

> **Note**: The `.conf` extension is required by systemd's OpenVPN service. Files with `.ovpn` extensions won't be recognized.

### Authentication Setup

Create a credentials file:

```bash
sudo nano /etc/openvpn/client/protonvpn-credentials.txt
```

Add two lines:
```
your_protonvpn_username
your_protonvpn_password
```

Secure the credentials file:

```bash
sudo chown openvpn:openvpn /etc/openvpn/client/protonvpn-credentials.txt
sudo chmod 600 /etc/openvpn/client/protonvpn-credentials.txt
```

Fix auth references in configs:

```bash
sudo sed -i 's|^auth-user-pass$|auth-user-pass /etc/openvpn/client/protonvpn-credentials.txt|' /etc/openvpn/client/*.conf
```

Verify the fix:

```bash
grep "auth-user-pass" /etc/openvpn/client/*.conf
```

Each line should show the full path to the credentials file.

### Systemd Timeout Configuration

OpenVPN connections can take time to establish. Increase the systemd timeout to prevent premature failures:

```bash
sudo mkdir -p /etc/systemd/system/openvpn-client@.service.d/
sudo tee /etc/systemd/system/openvpn-client@.service.d/override.conf <<EOF
[Service]
TimeoutStartSec=120
EOF
sudo systemctl daemon-reload
```

This gives OpenVPN 120 seconds to establish a connection.

## Script Implementation

### Project Structure

```
~/.config/waybar/
├── vpn-toggle.sh    # Manages VPN connections
└── vpn-status.sh    # Displays current status
```

### VPN Toggle Script

Create the toggle script:

```bash
mkdir -p ~/.config/waybar
touch ~/.config/waybar/vpn-toggle.sh
chmod +x ~/.config/waybar/vpn-toggle.sh
```

Complete script implementation:

```bash
#!/bin/bash
# VPN Toggle Script for ProtonVPN with Random Server Selection

CONFIGS=(
    "ca-free"
    "jp-free"
    "no-free"
    "mx-free-12.protonvpn.udp"
    "nl-free-135.protonvpn.udp"
    "pl-free-2.protonvpn.udp"
    "ro-free-28.protonvpn.udp"
    "sg-free-15.protonvpn.udp"
    "ch-free-4.protonvpn.udp"
)

STATE_FILE="/tmp/protonvpn-state"
CURRENT_FILE="/tmp/protonvpn-current"

# Map country codes to full names
get_country_name() {
    case "$1" in
        ca) echo "Canada" ;;
        jp) echo "Japan" ;;
        no) echo "Norway" ;;
        mx) echo "Mexico" ;;
        nl) echo "Netherlands" ;;
        pl) echo "Poland" ;;
        ro) echo "Romania" ;;
        sg) echo "Singapore" ;;
        ch) echo "Switzerland" ;;
        *) echo "Unknown" ;;
    esac
}

# Check if VPN is active
is_vpn_running() {
    systemctl is-active --quiet "openvpn-client@*"
}

# Stop all VPN connections
stop_vpn() {
    sudo systemctl stop "openvpn-client@*" 2>/dev/null
    rm -f "$STATE_FILE" "$CURRENT_FILE"
    sleep 0.5
    notify-send -t 3000 -u normal "VPN Disconnected" "Your connection is no longer protected"
}

# Start random VPN connection
start_vpn() {
    sudo systemctl stop "openvpn-client@*" 2>/dev/null
    sleep 1
    
    # Random server selection
    RANDOM_CONFIG="${CONFIGS[$RANDOM % ${#CONFIGS[@]}]}"
    COUNTRY_CODE=$(echo "$RANDOM_CONFIG" | cut -d'-' -f1)
    COUNTRY_NAME=$(get_country_name "$COUNTRY_CODE")
    
    notify-send -t 3000 -u normal "VPN Connecting" "Connecting to $COUNTRY_NAME..."
    sudo systemctl start "openvpn-client@${RANDOM_CONFIG}"
    sleep 3
    
    # Verify connection
    if systemctl is-active --quiet "openvpn-client@${RANDOM_CONFIG}"; then
        echo "connected" > "$STATE_FILE"
        echo "$RANDOM_CONFIG" > "$CURRENT_FILE"
        notify-send -t 3000 -u normal "VPN Connected" "Connected to $COUNTRY_NAME"
    else
        notify-send -t 3000 -u critical "VPN Failed" "Could not connect to $COUNTRY_NAME"
    fi
}

# Toggle logic
if is_vpn_running; then
    stop_vpn
else
    start_vpn
fi
```

#### Script Components

**CONFIGS Array**
: List of available VPN configurations. Config names must match `.conf` files in `/etc/openvpn/client/` (without the extension).

**State Files**
: Track current connection status and selected server for the status display script.

**get_country_name()**
: Converts country codes to full names for better user experience.

**is_vpn_running()**
: Checks if any OpenVPN service is active using systemd.

**stop_vpn()**
: Stops all VPN connections, cleans state files, and notifies the user.

**start_vpn()**
: Randomly selects a server, starts the connection, verifies success, and sends notifications.

### VPN Status Script

Create the status display script:

```bash
touch ~/.config/waybar/vpn-status.sh
chmod +x ~/.config/waybar/vpn-status.sh
```

Complete implementation:

```bash
#!/bin/bash
# VPN Status Script for Waybar

CURRENT_FILE="/tmp/protonvpn-current"

# Map country codes to full names
get_country_name() {
    case "$1" in
        ca) echo "Canada" ;;
        jp) echo "Japan" ;;
        no) echo "Norway" ;;
        mx) echo "Mexico" ;;
        nl) echo "Netherlands" ;;
        pl) echo "Poland" ;;
        ro) echo "Romania" ;;
        sg) echo "Singapore" ;;
        ch) echo "Switzerland" ;;
        *) echo "VPN" ;;
    esac
}

# Output JSON for Waybar
if systemctl is-active --quiet "openvpn-client@*"; then
    if [ -f "$CURRENT_FILE" ]; then
        CONFIG=$(cat "$CURRENT_FILE")
        COUNTRY_CODE=$(echo "$CONFIG" | cut -d'-' -f1)
        COUNTRY_NAME=$(get_country_name "$COUNTRY_CODE")
        echo "{\"text\":\" $COUNTRY_NAME\",\"tooltip\":\"VPN Connected: $CONFIG\",\"class\":\"connected\"}"
    else
        echo "{\"text\":\" Connected\",\"tooltip\":\"VPN Connected\",\"class\":\"connected\"}"
    fi
else
    echo "{\"text\":\"󰷷 Disconnected\",\"tooltip\":\"VPN Disconnected - Click to connect\",\"class\":\"disconnected\"}"
fi
```

The script outputs JSON with three fields:

| Field | Purpose |
|-------|---------|
| text | Display text in status bar |
| tooltip | Hover information |
| class | CSS class for styling |

## Waybar Integration

### Module Configuration

Edit `~/.config/waybar/config`:

```json
{
    "modules-right": [
        "custom/vpn",
        "pulseaudio",
        "clock"
    ],
    
    "custom/vpn": {
        "exec": "~/.config/waybar/vpn-status.sh",
        "interval": 5,
        "return-type": "json",
        "on-click": "~/.config/waybar/vpn-toggle.sh",
        "format": "{}"
    }
}
```

**Configuration Options**

| Option | Value | Description |
|--------|-------|-------------|
| exec | Script path | Status script location |
| interval | 5 | Update frequency (seconds) |
| return-type | json | Expected output format |
| on-click | Script path | Toggle script location |
| format | {} | Display format (uses JSON text field) |

### Module Styling

Add CSS to `~/.config/waybar/style.css`:

```css
#custom-vpn.connected {
    background-color: #a6e3a1;
    color: #1e1e2e;
    padding: 0 10px;
    margin: 0 5px;
    border-radius: 5px;
    font-weight: bold;
}

#custom-vpn.disconnected {
    background-color: #f38ba8;
    color: #1e1e2e;
    padding: 0 10px;
    margin: 0 5px;
    border-radius: 5px;
}
```

> **Note**: Colors are from Catppuccin theme. Adjust to match your color scheme.

## Sudo Configuration

Configure passwordless sudo for VPN management:

```bash
sudo visudo
```

Add this line (replace `your_username`):

```
your_username ALL=(ALL) NOPASSWD: /usr/bin/systemctl start openvpn-client@*, /usr/bin/systemctl stop openvpn-client@*
```

This restricts passwordless access to only these specific systemctl commands.

## Testing

### Manual Testing

Test toggle script:

```bash
~/.config/waybar/vpn-toggle.sh
```

You should see a notification and VPN should connect.

Verify connection:

```bash
systemctl status "openvpn-client@*"
```

Test status script:

```bash
~/.config/waybar/vpn-status.sh
```

Should output JSON with connection information.

### Waybar Integration Test

Restart Waybar:

```bash
killall waybar && waybar &
```

The VPN module should appear in your status bar. Click it to toggle the connection.

## Customization

### Adding Servers

Add config names to the `CONFIGS` array:

```bash
CONFIGS=(
    "ca-free"
    "us-free-01.protonvpn.udp"
    "de-free-15.protonvpn.udp"
    # Add more servers here
)
```

Update country mapping:

```bash
get_country_name() {
    case "$1" in
        # ... existing mappings
        us) echo "United States" ;;
        de) echo "Germany" ;;
        # Add more countries here
    esac
}
```

### Custom Notifications

Modify `notify-send` commands for custom icons:

```bash
notify-send -i network-vpn -t 3000 -u normal "VPN Connected" "Connected to $COUNTRY_NAME"
```

### Connection Logging

Add logging capability:

```bash
LOG_FILE="$HOME/.local/share/vpn-toggle.log"

# In start_vpn() after successful connection:
echo "$(date): Connected to $COUNTRY_NAME" >> "$LOG_FILE"

# In stop_vpn():
echo "$(date): Disconnected" >> "$LOG_FILE"
```

## Troubleshooting

### Connection Issues

Check OpenVPN logs:

```bash
sudo journalctl -u openvpn-client@* -f
```

### Authentication Failures

Verify credentials:

```bash
sudo cat /etc/openvpn/client/protonvpn-credentials.txt
ls -la /etc/openvpn/client/protonvpn-credentials.txt
```

### Timeout Problems

Increase timeout value in systemd override:

```bash
sudo nano /etc/systemd/system/openvpn-client@.service.d/override.conf
# Change TimeoutStartSec to 180 or higher
sudo systemctl daemon-reload
```

### Waybar Update Issues

Check script permissions:

```bash
ls -la ~/.config/waybar/vpn-*.sh
# Should show: -rwxr-xr-x
```

## Conclusion

This setup provides one-click VPN management directly from Waybar. The random server selection adds privacy benefits, and the visual feedback keeps you informed of your connection status.

Key features:
- **Simple**: ~100 lines of bash total
- **Reliable**: Uses systemd for process management
- **Flexible**: Easy to add/remove servers
- **Informative**: Desktop notifications and status display
- **Secure**: Passwordless sudo limited to specific commands

The entire system is lightweight, uses only standard Linux tools, and requires minimal maintenance once configured.

