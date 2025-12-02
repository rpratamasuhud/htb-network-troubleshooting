# HTB VPN MTU/MSS Troubleshooting Case Study

This document captures a real troubleshooting journey while working with Hack The Box (HTB) machines using OpenVPN. The issue initially appeared subtle: port scans and ICMP worked, but HTTP access consistently hung with no response. The investigation uncovered deeper network behavior related to MTU and MSS configurations.

---
## Table of Contents
- [Overview](#overview)
- [1. Background](#1-background)
- [2. Initial Observations](#2-initial-observations)
- [3. Clue #1 — The Early Redirect](#3-clue-1--the-early-redirect)
- [4. Clue #2 — Control Test with Another Machine](#4-clue-2--control-test-with-another-machine)
- [5. Root Cause — MTU and MSS Mismatch](#5-root-cause--mtu-and-mss-mismatch)
- [6. Fix — Manually Reduce MTU and MSS](#6-fix--manually-reduce-mtu-and-mss)
- [7. Why MTU and MSS Matter (Simplified)](#7-why-mtu-and-mss-matter-simplified)
- [8. Final Notes](#8-final-notes)
- [9. Recommended MTU Strategy](#9-recommended-mtu-strategy)
- [10. Conclusion](#10-conclusion)

---
## 1. Background

While connected to HTB via OpenVPN, the VPN tunnel established successfully:

- `ping` to target worked  
- `traceroute` worked  
- `nmap` revealed open ports (SSH 22, HTTP 80)  
- `curl` returned *partial* or *stuck* responses  
- Browser (Firefox/Chromium) hung indefinitely when visiting the same HTTP endpoint

At first glance, everything suggested the machine was alive and reachable. Yet HTTP simply would not load.

---

## 2. Initial Observations

### ✔ ICMP reachable  
```
ping 10.10.11.83
```

### ✔ Port 80 reachable  
```
nc -vz 10.10.11.83 80
```

### ✔ Nmap returned correct service info  
```
80/tcp open  http nginx 1.18.0
```

### ❌ Browser stuck loading HTTP  
### ❌ `curl` hung after sending the HTTP request

This indicated the TCP connection opened, the request was sent, but **the response never made it back**.

---

## 3. Clue #1 — The Early Redirect

A direct request to the IP revealed something interesting:

```
curl -v http://10.10.11.83
< HTTP/1.1 302 Moved Temporarily
< Location: http://previous.htb/
```

The machine immediately redirected to a hostname (`previous.htb`).  
After adding it to `/etc/hosts`, the next request looked fine:

```
curl -v http://previous.htb/
* Connected...
> GET /
* Request completely sent off
```

But then it froze. This suggested the request left your machine, the server responded, but the response packets were too large to pass through the VPN tunnel.

---

## 4. Clue #2 — Control Test with Another Machine

Using a different HTB machine (`10.129.x.x`) produced **a full HTML response**, proving that:

- curl works normally  
- The VPN connection itself works  
- Firefox/Chromium are not misconfigured  

Thus, the issue was **specific to the previous machine's response size and the tunnel path**.

This narrowed the problem down to **packet fragmentation inside the HTB VPN tunnel**.

---

## 5. Root Cause — MTU and MSS Mismatch

The VPN tunnel had an MTU that was **too large**, causing returning packets from port 80 to exceed what the tunnel could forward without fragmentation.

Because some VPN paths *block ICMP fragmentation negotiation*, this resulted in:

**➡ Large packets being dropped silently**  
**➡ Small packets (ping, small headers) working fine**  
**➡ Full HTTP responses failing**

This explains:

- why the HTTP request succeeded (small outgoing packet)  
- why the HTTP response froze (large incoming packet)

---

## 6. Fix — Manually Reduce MTU and MSS

Updating the `.ovpn` file resolved the issue:

```
tun-mtu 1300
mssfix 1200
```

After reconnecting:

### ✔ Browser worked  
### ✔ `curl` returned full HTML  
### ✔ No hanging HTTP requests  

A lower MTU allowed the tunnel to carry larger responses by ensuring packets along the route stayed within safe size limits.

---

## 7. Why MTU and MSS Matter (Simplified)

- **MTU (Maximum Transmission Unit):**  
  Maximum packet size allowed on a link.

- **MSS (Maximum Segment Size):**  
  Maximum payload size *inside* a TCP packet.

If MTU is too large for the VPN path, and ICMP is blocked, automatic fragmentation fails.

Reducing MTU (and optionally MSS) ensures packets fit safely through the tunnel.

###  MTU, MSS, and VPN Tunnel Interaction

                +-------------------------------------------------------+
                |                   Original Packet                     |
                |                                                       |
                |          [ IP Header | TCP Header |   Data   ]        |
                |                          (MSS)                        |
                +-------------------------------------------------------+
                                        |
                                        | Encapsulated inside VPN
                                        v
                +-------------------------------------------------------+
                |                    VPN Tunnel Packet                  |
                |                                                       |
                | [ VPN Header | IP Header | TCP Header |   Data   ]    |
                |                          (MSS)                        |
                +-------------------------------------------------------+
                                 ↑
                                 |
            If this entire packet exceeds **MTU**, it gets dropped,
            causing stalls, infinite loading, or connection hangs.


### What Happens When MTU Is Too Large (Simplified)

```text
Normal-sized packet (OK)
----------------------------------------
| VPN | IP | TCP | DATA........ | < MTU |
----------------------------------------

Oversized packet (Dropped)
-----------------------------------------------
| VPN | IP | TCP | DATA...................... |
-----------------------------------------------
               ^ Too large → exceeds MTU → dropped
```
---

## 8. Final Notes

This case highlights a scenario where:

- Connectivity appears normal  
- Basic tools (ping, nc) succeed  
- Service scans succeed  
- But **HTTP hangs indefinitely**

The cause is invisible at first glance: **VPN path MTU blackholing**.

By adjusting MTU/MSS manually, the entire HTTP pipeline becomes functional again.

This troubleshooting flow is valuable for real-world penetration testing and network debugging, especially when working over VPN tunnels or constrained network environments.

---

## 9. Recommended MTU Strategy

- Start with **1300**  
- If still stuck, test lower values (1200 → 1000)  
- MSS should generally be ~40 bytes smaller than MTU

Example:

| MTU | MSS (approx) |
|-----|--------------|
| 1300 | 1260 |
| 1200 | 1160 |
| 1000 | 960  |

Even if MSS is slightly higher, OpenVPN often auto-adjusts, but optimal pairing is recommended.

---

## 10. Conclusion

This troubleshooting journey shows how non-obvious connectivity issues arise from deep networking mechanics. With systematic testing, comparison, and hypothesis elimination, MTU/MSS misconfiguration was identified and resolved, restoring full access to the HTB machine.

This README acts both as documentation and as a learning reference for similar VPN and penetration-testing environments.

