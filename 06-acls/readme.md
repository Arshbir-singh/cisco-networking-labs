# Enterprise ACL Policy with DMZ Segmentation

A Cisco Packet Tracer lab implementing role-based network access control across three user VLANs and two server segments, with device management restricted to a single administrative host.

The focus of this project is **policy design and device hardening**, not just ACL syntax. Each filtering decision is documented with the reasoning behind it, including the trade-offs that were accepted and why.

---

## Skills Demonstrated

- Extended named ACLs with default-deny posture
- Standard ACLs applied to VTY lines (`access-class`)
- DMZ design and server segment separation
- Inter-VLAN routing (router-on-a-stick) at the access layer
- Single-area OSPF with passive-interface control
- Native VLAN mitigation against double-tagging
- SSHv2 device hardening across all managed infrastructure
- Structured verification and policy validation

---

## Topology

![Topology](docs/topology.png)

| Segment | Subnet | VLAN | Role |
|---|---|---|---|
| Sales | 192.168.1.0/24 | 10 | Restricted user LAN |
| HR | 192.168.2.0/24 | 20 | Internal user LAN |
| IT / Management | 192.168.99.0/24 | 99 | Administrative access |
| WAN link | 10.0.1.0/30 | — | R1 ↔ R2 |
| DMZ | 172.16.2.0/24 | — | Public-facing web server |
| Internal servers | 172.16.1.0/24 | — | File server |

Addressing convention: gateways at `.254`, infrastructure and hosts from the low end of each range.

### Device Addressing

| Device | Interface | Address |
|---|---|---|
| R1 | Gig0/1.10 | 192.168.1.254/24 |
| R1 | Gig0/1.20 | 192.168.2.254/24 |
| R1 | Gig0/1.99 | 192.168.99.254/24 |
| R1 | Gig0/0 | 10.0.1.1/30 |
| R1 | Loopback0 | 1.1.1.1/32 |
| R2 | Gig0/0 | 10.0.1.2/30 |
| R2 | Gig0/2 | 172.16.2.254/24 (DMZ) |
| R2 | Gig0/1 | 172.16.1.254/24 (internal) |
| R2 | Loopback0 | 2.2.2.2/32 |
| SW1 | VLAN 99 SVI | 192.168.99.2/24 |
| SW2 | VLAN 1 SVI | 172.16.2.2/24 |
| SW3 | VLAN 1 SVI | 172.16.1.2/24 |
| PC5 (admin) | Fa0 | 192.168.99.1/24 |
| Web server | Fa0 | 172.16.2.1/24 |
| File server | Fa0 | 172.16.1.1/24 |

---

## Access Policy

| From ↓ / To → | Sales | HR | IT | Web (DMZ) | File | Device CLI |
|---|---|---|---|---|---|---|
| **Sales** | — | ✗ | ✓ | HTTP + ICMP | ✗ | ✗ |
| **HR** | ✗ | — | ✓ | ✓ | ✓ | ✗ |
| **IT** | ✓ | ✓ | — | ✓ | ✓ | ✓ |

Sales is granted exactly what its business function requires — web access — and nothing else. HR retains access to both server segments. IT is unrestricted and PC5 is the only host permitted to reach any device command line.

---

## Design Decisions

**Servers are split across two subnets on separate physical interfaces.**
The web server sits in its own segment behind a dedicated R2 interface rather than sharing a subnet with the file server. This is what makes it a DMZ: an externally-reachable host physically separated from internal storage. It also simplifies the ACLs — policy can be written against whole subnets instead of individual host addresses, so the rules follow the network design rather than working around it.

**Router-on-a-stick is used at the access layer only.**
R1 carries three VLANs over a single trunk because the alternative would consume three physical ports for three user subnets. R2 has enough ports to give each server segment its own interface, so it does. Using a trunk where port count justifies it and separate interfaces where it doesn't is a deliberate distinction, not an inconsistency.

**ACLs are default-deny.**
Both traffic ACLs end in `deny ip any any`. Everything not explicitly permitted is dropped. This is a stronger posture than blocking named destinations and permitting the rest, and it makes every permit statement load-bearing — removing the ICMP line, for example, breaks ping to the web server while leaving HTTP working.

**ACLs are applied inbound at the source.**
Both are applied inbound on R1's user subinterfaces. Denied traffic is dropped at the first Layer 3 hop and never consumes the WAN link. Filtering at R2 instead would allow Sales→File traffic to cross the WAN before being discarded, and could not restrict Sales→HR at all, since that traffic never leaves R1.

**Sales and HR are mutually isolated rather than one-way.**
The original requirement was that Sales must not *initiate* to HR while HR could still reach Sales. Stateless ACLs cannot distinguish an initiating packet from a reply — a rule blocking Sales→HR also drops the return traffic of any session HR starts, so HR's connectivity breaks as a side effect. Achieving true one-way filtering requires stateful matching (`established` for TCP, `echo-reply` for ICMP) or a zone-based firewall. Mutual isolation was chosen as the cleaner and more common real-world outcome; the limitation is documented rather than hidden.

**Inbound ACLs also protect the router's own interfaces.**
Because `SALES_IN` and `HR_IN` are applied inbound and end in an explicit deny, they filter traffic addressed to R1 itself, not just traffic passing through it. Sales and HR hosts therefore cannot ping their own default gateways. This is intended behaviour — the router is not a permitted destination for those roles — and it is worth knowing during testing, where it can otherwise look like a misconfiguration.

**VLAN 99 is unrestricted.**
The administrative VLAN is unrestricted by design — that is what makes it the admin role. Its protection comes from `MNGMT_ACCESS` on every device's VTY lines, not from host-level filtering.

**The trunk native VLAN is an unused, empty VLAN.**
The R1–SW1 trunk uses native VLAN 1000, which exists in the VLAN database, has no member ports, and is excluded from the allowed VLAN list. All three data VLANs are tagged. Untagged frames arriving on the trunk have no valid destination, mitigating double-tagging VLAN hopping.

**OSPF is used rather than static routing.**
Single-area OSPF (area 0) advertises all user, server, and loopback networks across the WAN link. Every interface with no OSPF neighbour is set `passive-interface`, so hello packets are confined to the R1–R2 link. This prevents adjacency attempts on user-facing segments and keeps routing traffic off access ports.

---

## Configuration

### R1 — Traffic ACLs

```
ip access-list extended SALES_IN
 remark Sales: web server HTTP only, plus IT reachability
 permit tcp 192.168.1.0 0.0.0.255 host 172.16.2.1 eq www
 permit icmp 192.168.1.0 0.0.0.255 host 172.16.2.1
 permit ip 192.168.1.0 0.0.0.255 192.168.99.0 0.0.0.255
 deny ip any any

ip access-list extended HR_IN
 remark HR: both server subnets and IT, no Sales
 permit ip 192.168.2.0 0.0.0.255 172.16.1.0 0.0.0.255
 permit ip 192.168.2.0 0.0.0.255 172.16.2.0 0.0.0.255
 permit ip 192.168.2.0 0.0.0.255 192.168.99.0 0.0.0.255
 deny ip any any
```

Applied inbound on the user subinterfaces:

```
interface GigabitEthernet0/1.10
 ip access-group SALES_IN in

interface GigabitEthernet0/1.20
 ip access-group HR_IN in
```

### All Devices — Management ACL

Applied identically on R1, R2, SW1, SW2 and SW3:

```
ip access-list standard MNGMT_ACCESS
 permit host 192.168.99.1
 deny any

line vty 0 4
 access-class MNGMT_ACCESS in
 exec-timeout 5 0
 login local
 transport input ssh

line vty 5 15
 access-class MNGMT_ACCESS in
 exec-timeout 5 0
 login local
 transport input ssh
```

Access is restricted to a single administrative workstation rather than the whole IT subnet. Both VTY ranges are configured — `0 4` alone leaves `5 15` unprotected on platforms that support sixteen sessions.

### Device Hardening

Applied across all five devices:

| Control | Implementation |
|---|---|
| SSHv2 only | `ip ssh version 2`, `transport input ssh` |
| Local authentication | `username` + `login local` on VTY and console |
| Privileged access | `enable secret` (type 5 hash) |
| Idle session timeout | `exec-timeout 5 0` |
| Console protection | `login local`, `exec-timeout`, `logging synchronous` |
| Password storage | `service password-encryption` |
| Unused switch ports | Administratively shut down |
| Unused VLAN 1 (SW1) | SVI shut down; management moved to VLAN 99 |
| Unused router ports | `shutdown` on R1 Gig0/2 |

---

## Verification

Each user role was tested from its own workstation. Because the Packet Tracer command prompt retains scrollback, all tests from a single host are captured in one window — the images below show consecutive, uninterrupted runs rather than assembled fragments.

### Sales (PC1) — restricted

| # | Test | Expected |
|---|---|---|
| T2 | PC1 → web server, ping | Permit |
| T3 | PC1 → file server, ping | Deny |
| T4 | PC1 → PC4 (HR), ping | Deny |
| T13 | PC1 → SSH to SW2 | Deny |

![Sales policy enforcement](docs/screenshots/pc1-sales.png)

![Sales policy enforcement](docs/screenshots/pc1-sales-deny.png)

Denied traffic returns *destination host unreachable* rather than a timeout, because R1 actively rejects it at the ACL rather than silently dropping.

### HR (PC3) — internal access, no Sales

| # | Test | Expected |
|---|---|---|
| T5 | PC3 → PC1 (Sales), ping | Deny |
| T6 | PC3 → file server, ping | Permit |
| T7 | PC3 → web server, ping | Permit |
| T12 | PC3 → SSH to SW3 | Deny |

![HR policy enforcement](docs/screenshots/pc3-hr.png)

![HR policy enforcement](docs/screenshots/pc3-hr-ssh.png)

### IT (PC5) — unrestricted

| # | Test | Expected |
|---|---|---|
| T8 | PC5 → PC1, ping | Permit |
| T9 | PC5 → PC4, ping | Permit |
| T10 | PC5 → both servers, ping | Permit |
| T11 | PC5 → SSH to R1 | Permit |

![IT full access](docs/screenshots/pc5-it.png)
![IT full access](docs/screenshots/pc5-it-ssh.png)

### Web access (T1)

![Sales HTTP access to DMZ web server](docs/screenshots/pc1-http.png)

Sales reaches the web server over HTTP while every other destination is denied — the permit and the deny in the same policy.

### ACL hit counters

![R1 access-list counters](docs/screenshots/r1-access-lists.png)

Counters were cleared before the test sequence, so every match shown corresponds to a documented test above.

---

### Why T12 and T13 fail differently

Both are refused SSH attempts, but by different controls, and that distinction is the point of the management ACL.

HR has full IP reachability to 172.16.1.0/24, so its packets reach SW3 and are rejected by `MNGMT_ACCESS` on the VTY lines — the management ACL is the only thing stopping it. Sales never reaches SW2 at all, because `SALES_IN` permits only TCP/80 and ICMP toward the DMZ and drops the SSH attempt at R1. Two independent controls, verified separately.

Routing and Layer 2 state were confirmed with `show ip ospf neighbor`, `show ip route`, and `show interfaces trunk`.

---


## Known Limitations

**ACLs are stateless.** Filtering decisions are made per-packet with no awareness of session state, which is why one-way policy between Sales and HR was not achievable and mutual isolation was chosen instead. A production equivalent would use a zone-based policy firewall on R1 for true stateful inspection, permitting return traffic automatically while still blocking session initiation in one direction. That is the natural next iteration of this project.

**SW2 and SW3 are managed over VLAN 1.** SW1 shuts VLAN 1 and manages over VLAN 99, which is the correct pattern. SW2 and SW3 each serve a single server host and retain VLAN 1 management. In a production build all three would use a dedicated management VLAN.

---
## Project File
'acls.pkt'


