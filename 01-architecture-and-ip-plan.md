# 01 — Architecture, IP addressing and subnetting

## Hierarchical design

**Access layer:** 2960 switches connect PCs, laptops, servers and an IP phone. An access port typically carries one untagged data VLAN (and optionally a tagged voice VLAN).

**Core / collapsed distribution:** Two 3560 multilayer switches connect access switches and use SVIs for inter-VLAN routing. In a small lab they may provide both aggregation and core functions. The existence of two switches alone does **not** prove HSRP or failover; test both.

**WAN / edge:** HQ 2911 router reaches two branches by serial links and reaches the ASA on `10.0.1.0/30`. The firewall segments the ISP, inside, and DMZ networks.

**DMZ:** A separate ASA interface connects to a switch and server group. DMZ hosts should not have unrestricted access into the private network.

## IP addresses and their meaning

IPv4 is a 32-bit logical address. A subnet mask tells a device which bits identify a network versus a host. Example: `10.10.10.0/24` uses 24 network bits: mask `255.255.255.0`; usable typical hosts `10.10.10.1`–`10.10.10.254` (network `.0`, broadcast `.255`).

A `/30` (mask `255.255.255.252`) has four addresses total and traditionally two usable host addresses. It suits point-to-point infrastructure links: `10.0.1.0/30` is the network, `10.0.1.1` ASA, `10.0.1.2` HQ, `10.0.1.3` broadcast.

A **default gateway** is the router/SVI IP that an end device sends packets to when the destination lies outside its own subnet. A server on `192.168.200.0/24` would normally use `192.168.200.1` (the ASA DMZ interface), provided the server's own IP/mask are correctly configured.

## Verified routed networks

| Network | Purpose | How verified |
|---|---|---|
| `10.0.0.0/30`, HQ `10.0.0.2` | HQ-to-core link | HQ connected route |
| `10.0.0.4/30`, HQ `10.0.0.6` | HQ-to-core link | HQ connected route |
| `10.0.1.0/30`, HQ `.2`, ASA `.1` | HQ-to-ASA link | Both router and ASA output |
| `10.10.10.0/24` | Campus subnet | HQ OSPF route |
| `10.20.10.0/24` | Campus subnet | HQ OSPF route |
| `10.30.10.0/24` | Campus subnet | HQ OSPF route |
| `10.40.10.0/24` | Campus subnet | HQ OSPF route |
| `10.50.10.0/24` | Campus subnet | HQ OSPF route |
| `10.60.10.0/24` | Campus subnet | HQ OSPF route |
| `10.70.10.0/24` | Campus server subnet | HQ OSPF route |
| `172.16.1.0/30`, HQ `.2` | Branch 1 serial WAN | HQ connected route |
| `172.16.2.0/30`, HQ `.2` | Branch 2 serial WAN | HQ connected route |
| `172.16.10.0/24` | Branch 1 users | HQ OSPF route |
| `172.16.20.0/24` | Branch 2 users | HQ OSPF route |
| `192.168.200.0/24` | DMZ server subnet | ASA connected route |
| `203.0.113.0/30`, ASA `.2` | Simulated ISP link | ASA connected route |

## Intended VLAN mapping (verify on switches)

| VLAN ID | Department | Proposed/observed subnet association |
|---|---|---|
| 10 | IT | `10.10.10.0/24` |
| 20 | Sales | `10.20.10.0/24` |
| 30 | HR | `10.30.10.0/24` |
| 40 | Finance | `10.40.10.0/24` |
| 50 | Guest | `10.50.10.0/24` |
| 60 | Management | `10.60.10.0/24` |
| 70 | Servers / DHCP-DNS | `10.70.10.0/24` |

This mapping follows the previously described lab design, but **the exact VLAN-to-subnet mapping and SVI addresses must be confirmed** with `show vlan brief`, `show interfaces trunk`, and `show ip interface brief` on the core switches. Never guess the SVI gateway as `.1` without checking.

## Address types in the project

- `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`: RFC 1918 private addressing, not directly Internet-routable.
- `203.0.113.0/24`: reserved TEST-NET-3 documentation space; used here **only** to simulate an external link.
- Broadcast domains: each ordinary VLAN is a separate broadcast domain; routers and SVIs do not forward ordinary Layer 2 broadcasts between VLANs.
- Collision domains: each Ethernet switch port is a separate collision domain in classic terms; modern full-duplex switched Ethernet avoids normal CSMA/CD collisions.

## Key interview response

> I divided the campus into separate VLANs for departments, used /24 subnets for user segments, and /30 subnets for point-to-point router/firewall connections. The Layer 3 switches supply gateways for VLAN users; HQ connects the campus and branches, while ASA isolates private, DMZ and ISP-facing traffic.
