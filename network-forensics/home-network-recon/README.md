# Home Network Reconnaissance: Nmap Baseline Assessment

**Author:** Eric Sterman
**Date:** November 2026
**Category:** Network Reconnaissance / Asset Discovery
**Tools:** Kali Linux, Nmap 7.991, traceroute, VMware Workstation

> Step-by-step instructions for reproducing this lab are in [LAB_GUIDE.md](LAB_GUIDE.md).

---

## Executive Summary

I performed an authorized, non-intrusive reconnaissance assessment of my home network to build a security baseline: an inventory of every device, the services each router exposes, and how the network is laid out. A baseline is the starting point for incident response. Without knowing what "normal" looks like, it is hard to spot a new device, an unexpected open port, or a service that changed.

**Key findings:**

1. **Two separate home networks.** A wired network behind a Google Fiber router contains only my workstation and its router. A separate AT&T Wi-Fi network carries 19 devices, including phones, smart home devices, and appliances.
2. **Double NAT on the wired network.** Traceroute showed the wired router sits behind a second upstream router before reaching Google Fiber.
3. **Flat IoT network on the AT&T side.** Smart home devices (cameras, smart plugs, appliances, streaming devices) share one network with phones and a computer, with no segmentation.
4. **AT&T gateway exposes a small, mostly expected set of services**, but its admin interface uses an expired TLS certificate and a session cookie without the Secure flag, and its DNS service discloses its software version (Unbound 1.22.0), a version within the range affected by a published DNS hijack vulnerability.
5. **The wired network's router runs outdated, high-risk embedded software.** It has no standard web admin page, but it exposes a Boa web server (discontinued in 2005 and abused in real-world attacks) and two UPnP services, including one on port 52869, a port botnets have historically targeted on Realtek-based routers.

---

## Scope and Authorization

| In scope | Out of scope |
|---|---|
| Devices on my home LANs | Router public (WAN) IP addresses |
| LAN interfaces of my Google Fiber and AT&T routers | ISP infrastructure (traceroute hops beyond my routers) |
| My own computers and lab VMs | Brute-force, exploitation, or intrusive NSE scripts against routers |

All scanning was performed on networks and equipment I am authorized to test. Because the routers may be ISP-owned, testing was limited to non-intrusive discovery and enumeration.

---

## Lab Environment

| Component | Details |
|---|---|
| Host | MSI Aegis desktop, Windows 11 |
| Hypervisor | VMware Workstation |
| Scanning VM | Kali Linux 2026.2 |
| Kali adapters | eth0: NAT (internet), eth1: Host-only (isolated lab), eth2: Bridged (connected only during scans) |
| Isolated lab (not scanned) | Metasploitable2 and Windows VM on the Host-only network |

**Design decision:** I added a dedicated bridged adapter that stays disconnected by default. Kali joins the home network only when I connect it for a scan, which keeps the attack lab isolated the rest of the time. I also confirmed IP forwarding is disabled (`net.ipv4.ip_forward = 0`), so Kali cannot route traffic between the isolated lab and the home network.

### Network Diagram

```
                     Internet
                  /            \
        Google Fiber            AT&T
             |                    |
   Upstream router          AT&T Gateway (Humax)
             |              192.168.1.254
   Google Fiber router            |
   192.168.137.254          AT&T Wi-Fi: 192.168.1.0/24
   (double NAT)               19 devices: phones, Ring cameras,
             |                smart plugs, appliances, TVs,
   Wired LAN: 192.168.137.0/24   streaming devices, printer
     - MSI Aegis host
     - Kali VM (bridged)
```

---

## Methodology

1. **Setup:** configured a bridged adapter on Kali and verified it received an address on each target network (`ip a`, `ip route`).
2. **Path mapping:** used `traceroute` to identify the routers between Kali and the internet.
3. **Host discovery:** `nmap -sn` to find live hosts. On the local subnet, Nmap uses ARP, which returns MAC addresses and vendor names.
4. **Port scanning:** default top-1,000 TCP ports, all 65,535 TCP ports, and the top 20 UDP ports against each router.
5. **Enumeration:** service and version detection (`-sV`), OS fingerprinting (`-O`), and default safe scripts (`-sC`).
6. **Baseline:** saved combined results in all Nmap output formats (`-oA`) for future comparison with `ndiff`.

Raw output is in the [`scans/`](scans/) folder.

---

## Results

### Network 1: Wired (Google Fiber)

**Path to the internet** (`traceroute -i eth2 8.8.8.8`):

| Hop | Address | Notes |
|---|---|---|
| 1 | 192.168.137.254 | Google Fiber router (`router.local`) |
| 2 | 192.168.1.1 | Upstream router (second NAT layer) |
| 3–8 | * * * | No response (normal for ISP routers that drop traceroute probes) |
| 9 | Google Fiber network | ISP point of presence |
| 12 | 8.8.8.8 | Destination (Google DNS) |

**Host discovery** (`nmap -sn 192.168.137.0/24`):

| IP | Vendor | Device |
|---|---|---|
| 192.168.137.100 | Micro-Star Intl (MSI) | My workstation |
| 192.168.137.254 | ShenZhen ZhongRuiJing Technology | GFiber Wi-Fi 7 Router, model GRBE331C (vendor is the hardware manufacturer) |

Only two hosts responded besides Kali. The wired network has a very small attack surface.

### Network 2: AT&T Wi-Fi

**Host discovery** (`nmap -sn 192.168.1.0/24`): 19 hosts responded.

| Device category | Count | Vendors observed |
|---|---|---|
| Router | 1 | Humax (AT&T gateway, 192.168.1.254) |
| Workstation | 1 | Intel (my workstation's Wi-Fi adapter) |
| Phones and tablets | 3 | Apple; two devices with randomized private MAC addresses |
| Security cameras and doorbells | 4 | Ring |
| Smart home and IoT | 2 | Tuya Smart, Espressif (ESP32/ESP8266-based device) |
| Appliances | 1 | Whirlpool |
| TVs and streaming devices | 5 | Samsung, Amazon |
| Gaming | 1 | Nintendo |
| Printer | 1 | Hewlett Packard |

Two devices showed "Unknown" vendors because their MAC addresses are randomized (a privacy feature on modern phones and tablets).

### Router Comparison

| | Google Fiber router | AT&T gateway |
|---|---|---|
| IP address | 192.168.137.254 | 192.168.1.254 |
| Hardware vendor (MAC) | ShenZhen ZhongRuiJing | Humax |
| Open TCP ports | 24, 53, 10081, 49152, 52869 | 53, 80, 443, 2080, 51008 |
| Filtered TCP ports | None | 7547 (TR-069/CWMP), 10999, 49000, 51001, 51002, 51010, 61001 |
| UDP (top 20) | 53 open (DNS); open\|filtered: 67 (DHCP), 137, 139 (NetBIOS), 500 (ISAKMP), 631 (IPP), 1900 (UPnP/SSDP), 4500 (NAT-T), 49152; 11 closed | 53 open (DNS); open\|filtered: 67 (DHCP), 68, 137 (NetBIOS), 445, 520 (RIP), 631 (IPP), 1900 (UPnP/SSDP); 12 closed |
| Services and versions | Web (24): Boa HTTPd 0.94.14rc21; DNS: dnsmasq (version hidden); UPnP (49152): upnpd, fingerprint matches Cisco-Linksys E4200; UPnP (52869): MiniUPnP; 10081: unidentified (`famdc?`) | DNS: Unbound 1.22.0; Web: lighttpd 1.4.69 on four ports (80, 443, 2080, 51008) |
| OS guess | No exact match; TTL of 64 consistent with embedded Linux | No exact match; TTL of 64 and 1-hop distance consistent with an embedded Linux device |
| Admin interface (HTTP/HTTPS) | None on 80/443 (managed by app); Boa web server on port 24 returns `400 Bad Request` | Ports 80 and 443, redirect to `/cgi-bin/home.ha` |
| TLS certificate | N/A (no HTTPS service) | Issued to `dsldevice.domain_not_set.invalid`, organization Arris Group, Inc.; **expired May 29, 2025** |
| Behavior on closed ports | Rejects (TCP reset); 65,530 closed | Rejects (TCP reset); 65,523 closed, 7 filtered |

### Wired Router (192.168.137.254) Details

- **Identity:** The model label reads **GRBE331C**, which GFiber lists as its Wi-Fi 7 Router, provided as part of the service and managed through the gFiber app (GFiber, n.d.). The MAC vendor (ShenZhen ZhongRuiJing Technology) is the contract manufacturer. Nmap's UPnP fingerprint matched a Cisco-Linksys E4200, which is a **false match**, a good reminder that fingerprint-based identification needs physical or documentary confirmation.
- **Boa HTTPd 0.94.14rc21 (24/tcp):** Boa was discontinued in 2005 but remains embedded in many IoT devices and routers. Microsoft reported more than 1 million internet-exposed Boa components in 2022, and known Boa vulnerabilities (CVE-2017-9833, CVE-2021-33558) have been tied to attacks on critical infrastructure (Burt, 2022). The service returned `400 Bad Request` to the default script, so its purpose is unclear.
- **UPnP (49152 and 52869):** Two UPnP services are listening on the LAN. Port 52869 is the port used by the Realtek SDK `miniigd` UPnP daemon, whose SOAP interface had an unauthenticated command injection flaw (CVE-2014-8361) that botnets have mass-exploited (Ullrich, 2018). Nmap identified this service as MiniUPnP rather than Realtek's daemon, so CVE-2014-8361 is not confirmed to apply.
- **DNS (53):** dnsmasq. Unlike the AT&T gateway, it hides its version number (`bind.version: dnsmasq-`), which is good practice.
- **10081/tcp:** Found only in the full port scan. Version detection ran every probe for over two minutes and returned `famdc?`. The question mark means Nmap could not identify the service and fell back to the port-number guess. The port accepts connections but gives no recognizable response, which is typical of a proprietary vendor service (likely used by the router's management app or cloud connection).
- **No local web admin page:** Ports 80 and 443 are closed. The router appears to be managed through an app or cloud service instead.

### AT&T Gateway Details

- **DNS (53/tcp):** The `dns-nsid` script returned `id.server: dsldevice` and `bind.version: unbound 1.22.0`. The resolver advertises its exact software version to anyone on the LAN.
- **Web admin (80/443):** lighttpd 1.4.69. Both ports redirect to the AT&T management page at `/cgi-bin/home.ha`.
- **TLS certificate:** The certificate names Arris Group as the organization, showing the gateway firmware descends from Arris (Humax is the hardware manufacturer per the MAC address). The certificate expired on May 29, 2025.
- **Session cookie:** The `http-cookie-flags` script reported that the `SessionID` cookie does not have the Secure flag set even though HTTPS is in use.
- **Additional web services (2080, 51008):** Version detection identified both as HTTP served by the same lighttpd 1.4.69 instance. The basic scan had labeled 2080 as "autodesk-nlm," which was only a guess from the port number. These are likely internal management or device-communication endpoints, giving the gateway four web listeners in total.
- **UDP:** Only DNS (53) confirmed open. Several ports returned `open|filtered`, meaning no response was received, so Nmap cannot tell whether they are open or silently filtered. DHCP (67) is expected on a gateway. UPnP/SSDP (1900) is notable because UPnP lets devices on the LAN open inbound ports on the router automatically.
- **TR-069 (7547):** AT&T's remote management protocol is filtered from the LAN, which is expected. This port was widely targeted by Mirai-variant botnets in 2016, so filtering it is a positive control.
- **OS detection:** Nmap found no exact OS match. Note that `P=x86_64-pc-linux-gnu` in the fingerprint describes the scanning machine (Kali), not the target.

---

## Findings and Recommendations

| # | Finding | Risk | Recommendation |
|---|---|---|---|
| 1 | Smart home and IoT devices share the AT&T network with phones and a computer | Medium | Move IoT devices (cameras, plugs, appliances, TVs) to a separate guest or IoT network so a compromised device cannot reach personal devices. |
| 2 | Double NAT on the wired network | Low | No security issue (adds isolation), but can cause problems with VPNs, gaming, and port forwarding. Document it and keep it if those aren't needed. |
| 3 | Both the AT&T network and the Google Fiber upstream network use the 192.168.1.0/24 range | Low | Overlapping ranges can confuse troubleshooting and VPN routing. Consider changing one network's range. |
| 4 | AT&T gateway admin interface uses an expired, self-issued TLS certificate | Low | Users learn to click through browser certificate warnings, which makes a real man-in-the-middle warning easier to ignore. The certificate is ISP-managed; report it to AT&T, and access the admin page only from trusted devices. |
| 5 | AT&T admin `SessionID` cookie lacks the Secure flag while HTTP (port 80) is also open | Low–Medium | The session cookie could be sent over unencrypted HTTP and captured by someone on the LAN. Always use `https://192.168.1.254`, log out after use, and keep untrusted devices off this network (see Finding 1). |
| 6 | DNS service discloses its version (Unbound 1.22.0); upstream Unbound up to 1.24.1 is affected by CVE-2025-11411, a possible domain hijack via injected NS records | Low–Medium | Version disclosure helps attackers match known vulnerabilities. Vendors often backport fixes, so the version string alone does not prove the gateway is vulnerable. Keep gateway firmware updated (ISP-managed) and recheck the version on future scans. |
| 7 | Gateway runs four web listeners (80, 443, 2080, 51008), all lighttpd 1.4.69 | Low | Each listener is additional attack surface. Two are undocumented. Track them in future baseline scans and note any change. |
| 8 | UPnP/SSDP (1900/udp) may be active (`open\|filtered`) on a network with many IoT devices | Low–Medium | UPnP lets any LAN device open inbound ports without approval. Check the gateway's settings, disable UPnP if no device needs it, and review existing port forwards. |
| 9 | TR-069 remote management port (7547) filtered from the LAN | Positive | Expected and good. Confirms the ISP management interface is not exposed to local devices. |
| 10 | A current-generation ISP router (GFiber Wi-Fi 7, GRBE331C) still reports Boa HTTPd 0.94.14rc21, a web server discontinued in 2005 with known exploited vulnerabilities | Medium | The firmware is ISP-managed, so the owner cannot patch it directly. Keep automatic updates on in the gFiber app, report the finding to GFiber, and recheck the banner in future baseline scans. This is an example of the IoT supply chain risk Microsoft described (Burt, 2022). |
| 11 | Wired router exposes two UPnP services (49152, 52869), including a port historically targeted by botnets | Medium | Check the gFiber app for a UPnP setting and disable it if no device needs automatic port forwarding. Confirm with a rescan that 49152 and 52869 are closed. |
| 12 | Unidentified service on 10081/tcp, not in the top 1,000 ports | Low | Shows why a full port scan matters: a default scan would have missed it. Ask GFiber what the service is, and track it in future baselines. |
| 13 | Wired router hides its DNS software version | Positive | Good practice that limits information available to an attacker. |
| 14 | Wired router has no local web admin interface | Positive | Fewer exposed management services than the AT&T gateway. |

---

## Reflection

*[Answer each in a short paragraph after completing the port scans.]*

1. **Which router has the larger attack surface, and why?**

   It really depends on how an investigator would measure it. By the numbers, the AT&T gateway has a larger attack surface than the Google Fiber router. The AT&T scans showed four web listeners, DNS, a possible UPnP service, and an administrative page with an expired certificate. I also noticed a session cookie that wasn't protected. Together, these make a large and visible attack surface for an attacker. While the Google Fiber router exposes fewer services, the ones it runs are very risky: a Boa web server that was discontinued in 2005 and two UPnP services, including one on a port that botnets have historically targeted. Even though the Google Fiber router is brand new, it carries significant risk factors. In my assessment for this lab, I would rate the Google Fiber router as the higher risk because of the age and exploit history of its services, which matters more than the AT&T gateway's number of open ports.

2. **What surprised me about the devices or services on my network?**

   There were a few surprises from running this lab on my home networks. The first was the number of devices my family has placed on the AT&T network. The majority of them are smart home and IoT devices, sitting on the same flat network as our phones and my Mac desktop. I knew there were a good number of devices, but seeing them in the scan, listed by vendor, made the exposure of my home network real. The second surprise was the Google Fiber router itself. I got Google Fiber specifically for work and for conducting home labs, and this brand-new Wi-Fi 7 router is still running a web server from 2005. I assumed a new router would mean new software, but this lab showed me not to assume. Verify.

3. **What would I change to harden my home network?**

   First, I am going to disable UPnP on both routers so devices cannot open ports without my explicit approval. Second, I would move all of the smart home and IoT devices on the AT&T network to a separate guest or IoT network, so a compromised device cannot reach our phones or home computers. Third, since my wife is the one who manages the AT&T administrative page, I will make sure she only accesses it over HTTPS and logs out when she is finished. Finally, now that I have a baseline for both networks, I will conduct monthly scans and compare the results against it to identify any changes.

4. **How does a baseline support incident response and digital forensics?**

   In my 27 years as a criminal investigator, the questions I always asked in any case were what changed and when it changed. You have to establish a baseline before you can answer those questions. You have to know what normal looked like, whether that is a person's behavior or the state of a digital system. Capturing a baseline of a network works the same way. With a baseline established, a scan a month later will show whether a new device has appeared, a new port has opened, or a service version has changed. Digital forensic investigators compare evidence against known-good images and hashes for the same reason. This baseline gave me a documented, timestamped reference point for my home networks, which is exactly what needs to be in place before an incident occurs.

---

## Challenges and Lessons Learned

- **Bridged networking in VMware:** The bridged network (VMnet0) was missing from the Virtual Network Editor, so I added it and pinned it to a physical adapter. When I moved from wired to Wi-Fi, Kali got no IPv4 address until I re-pointed VMnet0 at the Wi-Fi adapter.
- **Shared folder for evidence handling:** Mounted a VMware shared folder in Kali and made it persistent with an `/etc/fstab` entry, using `nofail` so Kali still boots when the external drive is disconnected.
- **Discovering the network layout:** Initial host discovery found only two devices, even though phones were on Wi-Fi. Comparing IP addresses and gateways, then running traceroute, showed the home actually has two separate networks plus a double NAT.
- **Slow full-port scans:** My first full TCP scan accidentally targeted the entire /24 subnet instead of the gateway, which projected a 33-minute run and caused retransmission warnings from slower IoT devices. Narrowing the target to the gateway and adding `-T4 --min-rate 1000` finished in about 10 seconds. Lesson: subnet ranges belong in host discovery only; port scans should target specific hosts, especially around fragile IoT devices.
- **Scripts depend on service detection:** Running `-sC` alone against the wired router returned results only for DNS. Nmap picks default scripts based on the service name, and without `-sV` it didn't know ports 24, 49152, and 52869 were HTTP and UPnP, so no web or UPnP scripts ran. Combining `-sV -sC` (as in the baseline scan) produced the HTTP title for port 24.
- **Reading output carefully:** The SERVICE column in a basic port scan is only a guess from the port number. Version detection (`-sV`) is what identifies the actual software. Likewise, the `P=` field in an OS fingerprint describes the scanner, not the target.
- **Wrong scan type on the AT&T gateway:** My first "default" scan of the AT&T gateway was accidentally run with `-sn` (host discovery only), so it returned no port data. I removed that output and relied on the full TCP scan (`03_all-tcp_att.txt`), which covers all 65,535 ports, including the top 1,000 a default scan would check.
- **Redacting evidence for publication:** Before publishing, I masked all MAC addresses and device hostnames in the scan files using `sed`, and blurred them in screenshots. The unredacted originals remain in my lab environment, the same way original evidence is preserved while working copies are shared.

---

## References

Burt, J. (2022, November 23). Still using a discontinued Boa web server? Microsoft warns of supply chain attacks. *The Register*. https://www.theregister.com/2022/11/23/microsoft_boa_web_server/

GFiber. (n.d.). *Set up your gFiber Wi-Fi 7 router*. https://gfiber.com/support/en/answer/37741/

Ubuntu. (n.d.). *CVE-2025-11411*. Canonical. https://ubuntu.com/security/CVE-2025-11411

Ullrich, J. (2018, August 1). When cameras and routers attack phones: Spike in CVE-2014-8361 exploits against port 52869. *SANS Internet Storm Center*. https://isc.sans.edu/diary/23942

---

## Skills Demonstrated

- Network host discovery and asset inventory
- Route tracing and network topology mapping
- TCP and UDP port scanning
- Service, version, and OS enumeration
- Safe use of the Nmap Scripting Engine (default scripts)
- Baseline creation for change detection
- Linux administration: network interfaces, NetworkManager, file system mounts, `/etc/fstab`
- Virtual network design and lab isolation
- Scope definition, authorized testing, and technical reporting

## Next Steps

- Rescan monthly and compare against the baseline with `ndiff`.
- Segment IoT devices onto a separate network and rescan to verify.
- Repeat the methodology against Metasploitable2 on the isolated Host-only network, where intrusive and `vuln` scripts are safe to use.

---

## Repository Structure

```
home-network-recon-lab/
├── README.md        # This report
├── LAB_GUIDE.md     # Step-by-step lab instructions
├── scans/           # Nmap output files
└── screenshots/     # Evidence screenshots (MACs and identifying details redacted)
```
