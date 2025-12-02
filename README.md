# HTB VPN MTU/MSS Troubleshooting Case Study

This document presents a technical analysis of a network connectivity anomaly encountered on an HTB machine accessed through an OpenVPN tunnel. Although the host responded normally at the ICMP and service-scanning layers, HTTP traffic failed to load at the application layer. This case study outlines the observed symptoms, validation steps, and packet-level reasoning that led to the identification of an MTU/MSS mismatch as the root cause.

## Executive Summary
A connectivity anomaly was encountered  during an HTB assessment where HTTP traffic failed to load through an OpenVPN tunnel, despite the host being fully reachable at the ICMP, routing, and port-scanning layers. Initial tests indicated that TCP connections were established but stalled during data transfer, suggesting a packet-level transmission issue rather than an application-level failure.

Further investigation revealed that VPN traffic exceeded the available MTU, causing silent packet drops without triggering explicit error messages. Because TCP packet sizes were larger than the permitted path MTU, the handshake completed successfully while subsequent HTTP data transmission failed. Adjusting the OpenVPN parameters (`tun-mtu` and `mssfix`) restored stable data flow and resolved the issue.

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
$ ping -c 5 10.10.11.83
PING 10.10.11.83 (10.10.11.83) 56(84) bytes of data.
64 bytes from 10.10.11.83: icmp_seq=1 ttl=63 time=22.3 ms
64 bytes from 10.10.11.83: icmp_seq=2 ttl=63 time=22.0 ms
64 bytes from 10.10.11.83: icmp_seq=3 ttl=63 time=22.9 ms
64 bytes from 10.10.11.83: icmp_seq=4 ttl=63 time=22.9 ms
64 bytes from 10.10.11.83: icmp_seq=5 ttl=63 time=22.9 ms

--- 10.10.11.83 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4381ms
rtt min/avg/max/mdev = 21.999/22.585/22.908/0.381 ms

```

### ✔ Port 80 reachable  
```
$ nc -vz 10.10.11.83 80

previous.htb [10.10.11.83] 80 (http) open
```

### ✔ Nmap returned correct service info  
```
80/tcp open  http nginx 1.18.0
```

### ❌ Browser stuck loading HTTP  
<img width="918" height="717" alt="image" src="https://github.com/user-attachments/assets/488bfec9-a823-4042-ae74-b08da2751d16" />

### ❌ `curl` hung after sending the HTTP request
```bash
$ curl -v http://10.10.11.83

*   Trying 10.10.11.83:80...
* Connected to 10.10.11.83 (10.10.11.83) port 80
* using HTTP/1.x
> GET / HTTP/1.1
> Host: 10.10.11.83
> User-Agent: curl/8.15.0
> Accept: */*
> 
* Request completely sent off
< HTTP/1.1 302 Moved Temporarily
< Server: nginx/1.18.0 (Ubuntu)
< Date: Tue, 02 Dec 2025 16:25:37 GMT
< Content-Type: text/html
< Content-Length: 154
< Connection: keep-alive
< Location: http://previous.htb/
< 
<html>
<head><title>302 Found</title></head>
<body>
<center><h1>302 Found</h1></center>
<hr><center>nginx/1.18.0 (Ubuntu)</center>
</body>
</html>
* Connection #0 to host 10.10.11.83 left intact

```

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
```bash
$ curl -v 10.129.152.28     
*   Trying 10.129.152.28:80...
* Connected to 10.129.152.28 (10.129.152.28) port 80
* using HTTP/1.x
> GET / HTTP/1.1
> Host: 10.129.152.28
> User-Agent: curl/8.15.0
> Accept: */*
> 
* Request completely sent off
< HTTP/1.1 200 OK
< Date: Tue, 02 Dec 2025 16:58:26 GMT
< Server: Apache/2.4.38 (Debian)
< Vary: Accept-Encoding
< Content-Length: 4896
< Content-Type: text/html; charset=UTF-8
< 
<!DOCTYPE html>
<html lang="en">
<head>
        <title>Login</title>
        <meta charset="UTF-8">
          ...
</head>
<body>


   ...

</body>
</html>
* Connection #0 to host 10.129.152.28 left intact

```
- curl returned a complete HTML response on another HTB machine
- This confirms the server was working normally and the tunnel was capable of carrying full responses  

Thus, the issue suggest **specific to the previous machine's response size and the tunnel path**.

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
In short, the outgoing HTTP request succeeded because it was small, but the incoming
HTTP response exceeded the tunnel’s effective MTU. Since fragmentation negotiation
was blocked, oversized packets were silently dropped. This caused curl and browsers
to hang even though the server responded normally.

---

## 6. Fix — Manually Reduce MTU and MSS

To prevent oversized packets from exceeding the tunnel’s limits, the MTU and MSS
values were manually reduced inside the `.ovpn` configuration file:


```
tun-mtu 1300
mssfix 1200
```

This forces the VPN client to:
- send packets no larger than 1300 bytes, and  
- automatically clamp TCP MSS to a safe lower value.

After reconnecting the VPN:
- HTTP traffic started working immediately  
- curl successfully received full HTML responses  
- browsers (Firefox/Chromium) stopped hanging    

A lower MTU allowed the tunnel to carry larger responses by ensuring packets along the route stayed within safe size limits.

For quick testing, the issue can also be fixed without modifying the VPN configuration file, by manually adjusting the MTU on the active tunnel interface:

```bash
# Check current MTU
ip link show tun0

# Apply a lower MTU temporarily
sudo ip link set dev tun0 mtu 1000
```

This immediately allowed HTTP traffic to flow correctly, confirming that the failure was MTU-related.
However, this method is temporary and resets each time the VPN reconnects.
The permanent fix is still to update the MTU and MSS values in the .ovpn file.

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


