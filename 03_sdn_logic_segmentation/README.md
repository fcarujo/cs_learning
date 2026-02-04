# Project: Local Software-Defined Network & Security Lab

**Status:** Completed (Phase 1)
**Environment:** macOS (Apple Silicon M2), UTM Hypervisor, Linux (Ubuntu/Alpine)

## 1. Project Overview
This project simulates a **Software-Defined Network (SDN)** to replicate a corporate network environment locally. The goal was to manually build the "plumbing" of a secure network—routing, addressing, and segmentation—without relying on automated cloud VPC tools.

This lab demonstrates how to enforce network isolation and control traffic flow between an **Internal Network (LAN)** and the **Internet (WAN)** using a custom Linux Router.

## 2. Network Topology
Due to limitations in ARM64 virtualization (UTM VLAN bugs), logical segmentation was enforced via **Static Addressing** and **Routing Tables** on a shared virtual bus.

* **Host:** MacBook Air M2 (Hypervisor)
* **Zone A: The Gateway (Router)**
  * *OS:* Ubuntu Server
  * *WAN Interface:* `enp0s1` (DHCP from ISP)
  * *LAN Interface:* `enp0s2` (Static `192.168.10.1`)
  * *Role:* NAT, Masquerading, Firewall.
* **Zone B: The Target (Victim)**
  * *OS:* Alpine Linux
  * *IP:* `192.168.10.100` (Static)
  * *Role:* Vulnerable internal service (SSH enabled).
* **Zone C: The Threat Actor (Attacker)**
  * *OS:* Ubuntu Server (Kali-style setup)
  * *IP:* `192.168.10.200` (Static)
  * *Role:* Reconnaissance and Penetration Testing.

## 3. Implementation Details

### A. The "Plumbing" (Routing & NAT)
To allow the isolated LAN devices to access the internet without being directly exposed, I configured IP Masquerading (NAT) on the Router.

**Router Configuration (IP Forwarding):**
```bash
# Enabled packet forwarding in kernel
echo 1 > /proc/sys/net/ipv4/ip_forward

# Applied NAT masquerade rule (iptables)
iptables -t nat -A POSTROUTING -o enp0s1 -j MASQUERADE
```

### B. The Logic (Static Addressing)
Both the Victim and Attacker were manually configured to ignore the default DHCP and exist solely on the `192.168.10.x` subnet, using the Linux Router as their default gateway.

**Netplan Configuration (Attacker/Router):**
```yaml
network:
  ethernets:
    enp0s1:
      dhcp4: false
      addresses: [192.168.10.200/24]
      routes:
        - to: default
          via: 192.168.10.1
```

## 4. Security Assessment (Red Team)
Once connectivity was established, I performed an internal vulnerability assessment simulating an "Insider Threat" scenario.

* **Discovery:** Used `ping` to verify Layer 3 reachability.
* **Enumeration:** Used `nmap` to identify open ports.
  * *Command:* `nmap -sS 192.168.10.100`
  * *Result:* Discovered Port 22 (SSH) open.
* **Exploitation Validation:** Validated the vulnerability using `hydra` to brute-force the weak credential policy.

## 5. Challenges & Solutions
* **Issue:** UTM "Emulated VLAN" caused Layer 2 isolation failures (ARP requests failing).
* **Solution:** Migrated to "Shared Network" mode and enforced segmentation logically via subnetting and static routing. This mimicked "Zero Trust" principles where isolation is software-defined rather than physical.
* **Issue:** Alpine Linux (Victim) lost IP configuration on reboot.
* **Solution:** Identified the need for persistent configuration scripts (OpenRC) for production viability.

## 6. Future Scope
* **Hardening:** Implement `hosts.deny` and SSH Key-only authentication on the Victim.
* **Detection:** Install Snort IDS on the Router to detect the Nmap scans performed in Phase 1.
