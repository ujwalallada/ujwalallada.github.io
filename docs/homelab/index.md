# Homelab Projects

Welcome to my homelab documentation. Here I document infrastructure experiments, security implementations, and lessons learned from building a production-like home environment.

## Current Projects


#### [WireGuard VPN on Proxmox (LXC)](network/wireguard-proxmox-lxc.md)
Detailed troubleshooting of running WireGuard in Proxmox LXC containers, including packet-level analysis with tcpdump and architectural decision-making and it's failure

**Status:** LXC experiment complete → VM migration planned  
**Skills:** `WireGuard` `Proxmox` `Linux Networking` `tcpdump` `iptables` `NAT`

---

## Coming Soon

- **WireGuard VM Migration** - Moving VPN server to dedicated Opensense
- **pfSense Edge Firewall** - Enterprise firewall with VLANs and IDS/IPS

---

## Lab Environment

**Current Setup:**
- **Proxmox Host:** 192.168.1.10
- **LAN:** 192.168.1.0/24
- **WireGuard Subnet:** 10.8.0.0/24

**Planned Upgrades:**
- Intel i350-T4 quad-port NIC
- Opensense
---

## Learning Objectives

- [x] Proxmox LXC vs VM trade-offs
- [x] WireGuard protocol and configuration
- [x] Linux packet routing and NAT
- [x] Network troubleshooting with tcpdump
- [ ] pfSense advanced routing
- [ ] Infrastructure as Code (Ansible)
