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

Managing VPN connections through terminal commands gets old fast. You type the same commands over and over, manually pick servers, and lose track of whether you're connected or not. This guide walks through building a Waybar module that handles all of this with a single click.

The final result shows your current VPN status in your status bar, lets you toggle the connection with one click, randomly selects from your available servers, and sends desktop notifications when connections change. The whole thing is about 100 lines of bash split across two scripts.

## Prerequisites

You'll need a Linux system with systemd, a ProtonVPN account (free tier works fine), and Waybar already installed and configured. You should be comfortable with basic bash scripting and have root or sudo access for system configuration.

## Environment Setup

First, install OpenVPN and notification support. On Arch Linux, run `sudo pacman -S openvpn libnotify`. For Ubuntu or Debian, use `sudo apt install openvpn libnotify-bin`. Fedora users need `sudo dnf install openvpn libnotify`.

### Getting ProtonVPN Configuration Files

Head to account.protonvpn.com and go to Downloads, then OpenVPN configuration files. Pick the servers you want to use and download them. You can grab as many as you want - the script will randomly select from whatever you provide.

Move those downloaded configs to the OpenVPN directory with `sudo mv ~/Downloads/*.ovpn /etc/openvpn/client/`. Then rename them from .ovpn to .conf, which systemd requires. Change to the directory with `cd /etc/openvpn/client/` and run this loop:

```bash
for file in *.ovpn; do
    sudo mv "$file" "${file%.ovpn}.conf"
done
```

The .conf extension isn't optional. Systemd's OpenVPN service won't recognize .ovpn files.

### Setting Up Authentication

Create a credentials file at `/etc/openvpn/client/protonvpn-credentials.txt` with your ProtonVPN username on the first line and password on the second line. Lock down the permissions with `sudo chown openvpn:openvpn /etc/openvpn/client/protonvpn-credentials.txt` followed by `sudo chmod 600 /etc/openvpn/client/protonvpn-credentials.txt`.

The downloaded config files reference auth-user-pass without specifying the credentials file path. Fix that with this sed command:

```bash
sudo sed -i 's|^auth-user-pass$|auth-user-pass /etc/openvpn/client/protonvpn-credentials.txt|' /etc/openvpn/client/*.conf
```

Verify it worked by running `grep "auth-user-pass" /etc/openvpn/client/*.conf`. Each line should show the full path to your credentials file.

### Adjusting Systemd Timeout

OpenVPN connections can take a while to establish, especially on slower networks. The default systemd timeout is too aggressive and will kill connections before they finish establishing. Create an override to bump the timeout to 120 seconds:

```bash
sudo mkdir -p /etc/systemd/system/openvpn-client@.service.d/
sudo tee /etc/systemd/system/openvpn-client@.service.d/override.conf <<EOF
[Service]
TimeoutStartSec=120
EOF
sudo systemctl daemon-reload
```

## Script Implementation

The setup uses two scripts in your Waybar config directory. The toggle script manages connections and the status script displays current state.

### VPN Toggle Script

Create the file at `~/.config/waybar/vpn-toggle.sh` and make it executable with `chmod +x ~/.config/waybar/vpn-toggle.sh`. Here's the complete script:

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

The CONFIGS array holds your available VPN configurations. These names must match the .conf files in `/etc/openvpn/client/` without the extension. The script uses two temporary files to track connection state and which server is currently active. This information gets passed to the status script.

The get_country_name function converts two-letter country codes to full names for notifications. Add more cases here if you're using servers from additional countries.

When you click the Waybar module, is_vpn_running checks if any OpenVPN service is active. If something is running, stop_vpn kills all connections and cleans up the state files. If nothing is running, start_vpn picks a random server from your array and attempts to connect.

The start_vpn function does a few things. It stops any existing connections first (in case something is stuck), waits a second for cleanup, then randomly selects a server. It extracts the country code from the config name, looks up the full country name, and sends a notification that connection is starting.

After starting the systemd service, the script waits three seconds and verifies the connection actually established. If successful, it writes to the state files and sends a success notification. If the connection failed, you get an error notification instead.

### VPN Status Script

Create `~/.config/waybar/vpn-status.sh` and make it executable. This script runs every few seconds and outputs JSON that Waybar displays:

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

The script checks if any OpenVPN service is active. If it finds one, it reads the current config from the state file, extracts the country code, looks up the country name, and outputs JSON with that information. The JSON includes text to display in the bar, tooltip text that appears on hover, and a CSS class for styling.

If no VPN is running, it outputs disconnected state with different text and class. Waybar parses this JSON and renders it accordingly.

## Waybar Integration

Open your Waybar config at `~/.config/waybar/config` and add the custom VPN module:

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

The exec parameter points to your status script. Waybar runs this script every 5 seconds (controlled by interval) and expects JSON output (return-type). When you click the module, it runs the toggle script. The format field uses the text value from the JSON output.

### Styling the Module

Add some CSS to `~/.config/waybar/style.css` to differentiate connected and disconnected states:

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

These colors come from the Catppuccin theme. Adjust them to match your color scheme. The connected class uses green, disconnected uses red. Both get some padding and rounded corners to stand out in the bar.

## Sudo Configuration

The toggle script needs to start and stop systemd services, which requires root privileges. You could run the script with sudo and type your password every time, but that defeats the purpose of a one-click toggle.

Configure passwordless sudo for just these specific commands. Run `sudo visudo` and add this line, replacing your_username with your actual username:

```
your_username ALL=(ALL) NOPASSWD: /usr/bin/systemctl start openvpn-client@*, /usr/bin/systemctl stop openvpn-client@*
```

This grants passwordless sudo access only for starting and stopping OpenVPN client services. Nothing else gets elevated privileges.

## Testing

Before integrating with Waybar, test the toggle script directly. Run `~/.config/waybar/vpn-toggle.sh` from a terminal. You should see a notification that the VPN is connecting, followed by another notification when it connects successfully.

Verify the connection is actually active with `systemctl status "openvpn-client@*"`. You should see an active service for whichever server the script randomly selected.

Test the status script with `~/.config/waybar/vpn-status.sh`. It should output JSON showing your connected state and country.

Restart Waybar to load the new module with `killall waybar && waybar &`. The VPN module should appear in your bar showing current status. Click it to toggle the connection. You should see notifications and the status should update within 5 seconds.

## Customization

Adding more servers is straightforward. Just add config names to the CONFIGS array and update the country mapping function with any new country codes. The config names must match your .conf files in `/etc/openvpn/client/` without the extension.

If you want custom notification icons, modify the notify-send commands to include `-i network-vpn` or whatever icon name you prefer. You can find available icon names with `ls /usr/share/icons/*/status/`.

For connection logging, add a log file variable like `LOG_FILE="$HOME/.local/share/vpn-toggle.log"` at the top of the toggle script. Then append log entries after successful connections with `echo "$(date): Connected to $COUNTRY_NAME" >> "$LOG_FILE"` and similar for disconnections.

## Troubleshooting

If connections aren't establishing, check the OpenVPN logs with `sudo journalctl -u openvpn-client@* -f`. This shows real-time log output and will reveal authentication problems, network issues, or configuration errors.

Authentication failures usually mean wrong credentials or incorrect file paths in the config files. Verify your credentials file with `sudo cat /etc/openvpn/client/protonvpn-credentials.txt` and check permissions with `ls -la /etc/openvpn/client/protonvpn-credentials.txt`. The file should be owned by openvpn:openvpn with 600 permissions.

If connections timeout, increase the systemd timeout value in the override file. Edit `/etc/systemd/system/openvpn-client@.service.d/override.conf` and change TimeoutStartSec to 180 or higher. Run `sudo systemctl daemon-reload` after changing it.

When the Waybar module isn't updating, check that both scripts have execute permissions with `ls -la ~/.config/waybar/vpn-*.sh`. You should see `-rwxr-xr-x` for both files.

## Conclusion

This setup gives you one-click VPN management without leaving your desktop environment. The random server selection adds a privacy benefit by not always connecting to the same location, and desktop notifications keep you informed without having to check the status bar constantly.

The whole system uses standard Linux tools and requires minimal maintenance. Add or remove servers by editing one array. Change styling by editing CSS. The scripts are simple enough to modify for different VPN providers if you're using something other than ProtonVPN - just adjust the config file locations and names accordingly.
