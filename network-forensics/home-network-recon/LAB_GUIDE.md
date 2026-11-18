# Lab Guide: Home Network Reconnaissance with Nmap

**Author:** Eric [Last Name]
**Date:** [Date completed]
**Category:** Network Reconnaissance / Asset Discovery
**Tools:** Kali Linux, Nmap, ndiff, VMware Workstation

---

## Overview

This lab builds a security baseline of my home network using Nmap. I discover active hosts, enumerate open ports and services on two home routers (Google Fiber and AT&T), fingerprint operating systems, and document findings the same way I would for a real assessment.

A baseline answers a basic question in incident response: **what does "normal" look like?** Without one, it is hard to spot a new device, an unexpected open port, or a service that changed.

## Objectives

- Configure a Kali VM to safely access the home network.
- Discover live hosts on the network.
- Enumerate open ports, services, and versions on both routers.
- Fingerprint router operating systems.
- Compare the two routers' attack surfaces.
- Save results in reusable formats and practice change detection with `ndiff`.
- Write findings and recommendations in a professional format.

## Scope and Authorization

| In scope | Out of scope |
|---|---|
| Devices on my own home LAN | Router public (WAN) IP addresses |
| LAN interface of my Google Fiber router | ISP networks upstream of my routers |
| LAN interface of my AT&T router | Any device I do not own or have permission to scan |
| My own computers and lab VMs | Brute-force, exploitation, or intrusive NSE scripts against routers |

All scanning was performed on networks and equipment I am authorized to test. The routers may be ISP-owned equipment, so testing was limited to non-intrusive discovery and enumeration.

## Lab Environment

| Component | Details |
|---|---|
| Host | MSI Aegis desktop, Windows |
| Hypervisor | VMware Workstation |
| Attacker VM | Kali Linux (Adapter 1: NAT, switched to Bridged for this lab; Adapter 2: Host-only) |
| Target 1 | Google Fiber router: `[IP]` |
| Target 2 | AT&T router: `[IP]` |
| Isolated lab (not used here) | Metasploitable2 and Windows VM on Host-only network |

### Network Diagram

```
        Internet
            |
    [AT&T Router]  [IP]
            |
  [Google Fiber Router]  [IP]
            |
     Home LAN (192.168.x.0/24)
       |            |
  MSI Aegis host   Other home devices
       |
  Kali VM (Bridged)
```

*Update the diagram to match how your routers are actually connected.*

---

## Part 1: Setup

### 1.1 Find the router addresses

On the Windows host:

```
ipconfig
```

Record the **Default Gateway**. Then trace the path out to the internet to find the second router:

```
tracert 8.8.8.8
```

The first one or two private IPs (192.168.x.x, 10.x.x.x, 172.16-31.x.x) are your routers.

| Router | IP address |
|---|---|
| Google Fiber | |
| AT&T | |

### 1.2 Switch Kali to Bridged mode

1. Shut down the Kali VM.
2. VMware Workstation → Kali → Edit virtual machine settings → **Network Adapter** → select **Bridged: Connected directly to the physical network**.
3. Leave Network Adapter 2 on Host-only.
4. If Kali doesn't get a home network IP, open Edit → Virtual Network Editor → Change Settings → select **VMnet0** → change "Bridged to" from Automatic to your physical Ethernet or Wi-Fi adapter.
5. Boot Kali and confirm it has an address on your home network:

```
ip a
ip route
```

**Screenshot:** `screenshots/01_bridged_ip.png`

### 1.3 Update Kali and Nmap

```
sudo apt update && sudo apt full-upgrade -y
nmap --version
```

Record the Nmap version: `______`

---

## Part 2: Host Discovery

Find every live device on the network.

```
sudo nmap -sn 192.168.x.0/24 -oN scans/01_host_discovery.txt
```

**Questions**
1. How many hosts responded?
2. Can you identify each device by its MAC vendor (Google, Apple, Samsung, and so on)?
3. Are there any devices you don't recognize?

**Screenshot:** `screenshots/02_host_discovery.png`

---

## Part 3: Port Scanning

Run each scan against **both** routers. Replace `[ROUTER_IP]` and `[name]` each time.

### 3.1 Default scan (top 1,000 TCP ports)

```
sudo nmap [ROUTER_IP] -oN scans/02_default_[name].txt
```

### 3.2 Full TCP port scan

```
sudo nmap -p- [ROUTER_IP] -oN scans/03_all_tcp_[name].txt
```

### 3.3 Top UDP ports

```
sudo nmap -sU --top-ports 20 [ROUTER_IP] -oN scans/04_udp_[name].txt
```

**Questions**
1. Did the full TCP scan find anything the default scan missed?
2. Which UDP ports showed `open` vs. `open|filtered`? Why is UDP harder to scan accurately?

---

## Part 4: Service and OS Enumeration

### 4.1 Service and version detection

```
sudo nmap -sV [ROUTER_IP] -oN scans/05_versions_[name].txt
```

### 4.2 OS fingerprinting

```
sudo nmap -O [ROUTER_IP] -oN scans/06_os_[name].txt
```

### 4.3 Default (safe) scripts

```
sudo nmap -sC [ROUTER_IP] -oN scans/07_scripts_[name].txt
```

**Questions**
1. What service and version is running on each open port?
2. How confident was Nmap's OS guess? Does it match what you know about the device?
3. What did the default scripts reveal (page titles, SSL certificate details, DNS info)?

**Screenshot:** `screenshots/03_service_scan.png`

---

## Part 5: Baseline and Change Detection

### 5.1 Save a full baseline in all formats

```
sudo nmap -sV -sC -O [ROUTER_IP] -oA scans/baseline_[name]
```

This creates `.nmap` (readable), `.xml` (for tools), and `.gnmap` (for grep) files.

### 5.2 Compare against a later scan

Rescan after a week or a month, then compare:

```
sudo nmap -sV -sC -O [ROUTER_IP] -oA scans/rescan_[name]
ndiff scans/baseline_[name].xml scans/rescan_[name].xml
```

**Questions**
1. Did anything change between scans?
2. How would a change like this help detect a compromised device in a real network?

---

## Part 6: Findings

### Router Comparison

| | Google Fiber | AT&T |
|---|---|---|
| IP address | | |
| Open TCP ports | | |
| Open UDP ports | | |
| Services and versions | | |
| OS guess | | |
| Admin interface (HTTP/HTTPS) | | |

### Findings Table

| # | Device | Port/Service | Observation | Risk | Recommendation |
|---|---|---|---|---|---|
| 1 | | | | Low / Med / High | |
| 2 | | | | | |
| 3 | | | | | |

**Example entry:** *Port 80/tcp, HTTP admin page available without encryption. Risk: Low (LAN only). Recommendation: use HTTPS for the admin page if supported; confirm remote management is disabled.*

---

## Part 7: Reflection

Answer in a short paragraph each:

1. Which router has the larger attack surface, and why?
2. What surprised you about the devices or services on your network?
3. What changes would you make to harden your home network based on these results?
4. How does building a baseline support incident response and digital forensics?

---

## Skills Demonstrated

- Network host discovery and asset inventory
- TCP and UDP port scanning
- Service, version, and OS enumeration
- Safe use of Nmap Scripting Engine (NSE) default scripts
- Baseline creation and change detection with `ndiff`
- Scope definition and authorized testing practices
- Technical documentation and risk-based recommendations

## Next Steps

- Repeat the full methodology against Metasploitable2 on the Host-only network, where intrusive and `vuln` scripts are safe to use.
- Import the XML results into a tool such as Zenmap or a spreadsheet for visualization.
- Schedule recurring scans and track changes over time.

## Repository Structure

```
home-network-recon-lab/
├── README.md
├── scans/          # Nmap output files
└── screenshots/    # Evidence screenshots
```

> **Privacy note:** Before pushing to GitHub, redact MAC addresses, public IPs, device names that identify family members, and Wi-Fi network names from screenshots and scan files.
