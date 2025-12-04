# 🔐 WireGuard VPN Setup on Debian VM

A documentation of my attempt to set up WireGuard VPN on a Debian VM, the challenges I encountered, and why I'm moving to an OPNsense-based solution.

---

## Overview

**Goal:** Set up a WireGuard VPN to securely access my home network from anywhere.

**Environment:**
- Proxmox Host (static): **192.168.1.10**
- Debian VPN server (LXC, attempt 1): **192.168.1.171**
- Home network with dynamic IPv6
- Mobile and laptop clients

**Outcome:** Approach failed due to ISP router limitations. Switching to OPNsense firewall solution.

---

## System Architecture (Attempted)

```
Internet
   │
   └─ ISP Router (192.168.1.1)
        ├─ Proxmox Host
        │    └─ Debian VM (192.168.1.171)
        │         └─ WireGuard Server (port 51820)
        │
        └─ Clients (mobile, laptop)
```

---

## Setup Process

### 1. Install WireGuard

On the Debian VM:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install wireguard wireguard-tools curl -y
```

### 2. Enable IP Forwarding

Create `/etc/sysctl.d/99-sysctl.conf` and add:

```ini
net.ipv4.ip_forward=1
```

Apply with:
```bash
sudo sysctl --system
```

### 3. Generate Keys

For the server and each client:

```bash
cd /etc/wireguard
sudo umask 077

# Server keys
sudo wg genkey | sudo tee server_private.key | wg pubkey | sudo tee server_public.key

# Mobile client keys
sudo wg genkey | sudo tee mobile_private.key | wg pubkey | sudo tee mobile_public.key
```

### 4. Configure the Server

Create `/etc/wireguard/wg0.conf`:

```ini
[Interface]
PrivateKey = <SERVER_PRIVATE_KEY>
Address = 10.8.0.1/24
ListenPort = 51820

# NAT rules - replace ens18 with your interface name
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o ens18 -j MASQUERADE

[Peer]    # Mobile Client
PublicKey = <MOBILE_PUBLIC_KEY>
AllowedIPs = 10.8.0.2/32
PersistentKeepalive = 25
```

Start WireGuard:
```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

### 5. Configure Firewall

Set up UFW to allow WireGuard:

```bash
sudo ufw allow 22/tcp      # SSH
sudo ufw allow 51820/udp   # WireGuard
sudo ufw enable
```

### 6. Configure Clients

**Mobile (Android):**

Create a config file:

```ini
[Interface]
PrivateKey = <MOBILE_PRIVATE_KEY>
Address = 10.8.0.2/32
DNS = 1.1.1.1

[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
Endpoint = <YOUR_IPV6_OR_DOMAIN>:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

Import into the WireGuard app (via QR code or manual entry).

---

## The Challenges I Faced

### Challenge 1: Router Firewall Blocking VPN

**The Problem:**
My router's firewall is set to "High" security mode, blocking all incoming VPN connections. While I *can* disable the firewall, **I choose not to** due to serious security implications (explained below).

**What I Tried:**

1. **Port Forwarding Configuration**
   - Correctly set up UDP 51820 → 192.168.1.171
   - Verified multiple times, rebooted router

2. **Firewall Security Levels**
   - Tried Medium and Low security settings
   - Still blocked incoming VPN traffic

3. **Tested with Firewall Disabled**
   - Temporarily disabled firewall to test
   - VPN worked perfectly during this time

### Challenge 2: IPv6 + DuckDNS Approach

**The Attempt:**
I noticed that my ISP is providing IPV6 address (public) and this gets updated after a certain period of time, hence I tried using IPv6 with DuckDNS for dynamic DNS.

Here I learnt the concept of CGNAT and why does ISPs use this.

*Brief Info (Please skip if you already knew this)
**What is CGNAT?**
Carrier-Grade NAT is a technique ISPs use when they run out of public IPv4 addresses. Instead of giving each customer a unique public IP, they share one public IP among many customers.
**Why ISPs Use It:**
- IPv4 address exhaustion (not enough IPs for everyone)*

**DuckDNS Setup:**

DuckDNS is a free service that gives you a permanent domain name that automatically updates to point to your current IP.

Setup on server:

```bash
mkdir ~/duckdns
cd ~/duckdns
nano duck.sh
```

Script content:
```bash
#!/bin/bash
IPV6=$(curl -6 -s ifconfig.me)
echo url="https://www.duckdns.org/update?domains=YOUR-SUBDOMAIN&token=YOUR-TOKEN&ipv6=$IPV6&clear=true" | curl -k -o ~/duckdns/duck.log -K -
```

**Important:** The `&clear=true` parameter removes the IPv4 address from DNS, forcing clients to use IPv6 only.

Set to auto-update every 5 minutes:
```bash
chmod +x duck.sh
crontab -e
```

Add:
```
*/5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1
```

**Cron Syntax Explained:**
- `*/5` = Every 5 minutes
- `* * * *` = Every hour, day, month, and day of week
- `>/dev/null 2>&1` = Discard output to avoid email spam

**Why This Failed:**

Despite having IPv6 and DuckDNS configured:
- VPN worked when firewall was disabled (testing only)
- With firewall re-enabled at any security level, all connections blocked
- **The router's firewall blocks incoming traffic regardless of IPv4 or IPv6**
- DuckDNS correctly updated addresses, but traffic never reached my server

**Lesson Learned:** IPv6 bypasses NAT/port forwarding complexity, but it doesn't bypass firewall rules. Whether using IPv4 or IPv6, if the firewall blocks incoming connections on port 51820, the VPN won't work.

---

**The Core Problem:** While I *could* disable the router's firewall to make VPN work, I'm **unwilling to compromise security**. I need a solution that provides both VPN access and proper network protection.

---

## The Solution: Moving to OPNsense

### Why OPNsense?

Since I choose not to disable the router's firewall for security reasons, I need to place my own firewall/router **in front** of my network that I have full control over - one that can properly allow VPN traffic while maintaining security.

## Conclusion

While the WireGuard setup on Debian VM was technically sound, **the router's firewall blocking incoming connections made it unviable** without disabling security features. Since I'm unwilling to compromise network security by disabling the firewall, I need a better architectural solution.

This experience taught me valuable lessons about network architecture, ISP limitations, and the importance of proper infrastructure. The time spent troubleshooting wasn't wasted - understanding *why* this approach failed will inform better decisions in the OPNsense implementation.

---

**Status:** Failed Approach - Moving to OPNsense  
