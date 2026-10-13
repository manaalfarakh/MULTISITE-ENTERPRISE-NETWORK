# 06 — Validation plan and interview revision questions

## Evidence checklist

- [x] HQ `show ip route` confirms default to ASA `10.0.1.1`.
- [x] HQ `show ip route` confirms OSPF routes to seven campus subnets.
- [x] HQ `show ip route` confirms OSPF routes to both branches.
- [x] ASA `show run interface` confirms three named zones and IPs.
- [ ] Confirm ISP next hop `203.0.113.1` (assumed, not proven).
- [ ] Resolve/explain unusual ASA `10.0.0.0/30` connected route.
- [ ] `show route` on ASA confirms proper return routes/default.
- [ ] `show nat` confirms both outbound PAT rules actually installed.
- [ ] One client from each campus VLAN reaches a test ISP endpoint.
- [ ] Clients in both branches reach the ISP test endpoint.
- [ ] Internal browser can load the actual DMZ web server by IP.
- [ ] Internal DNS queries resolve test hostname(s).
- [ ] Test mail flow if MAIL Server is configured.
- [ ] Outside→DMZ access works only on allowed published services.
- [ ] DMZ→INSIDE unsolicited traffic is blocked by policy.
- [ ] `show standby brief` proves correct HSRP active/standby roles and failover.
- [ ] `show spanning-tree` confirms root and loop-free access paths.
- [ ] Explicit guest/HR/Finance segmentation ACL tests.
- [ ] Access-port security configuration and violation tests.
- [ ] Save final Packet Tracer project and sanitized configs/screenshots.

## Diagnostic order for a broken ping

1. Verify source/destination actual addresses and mask.
2. Verify interface status and cabling.
3. Same subnet? ARP/MAC table. Different subnet? Default gateway + SVI.
4. Verify outward route (`show ip route` / `show route`).
5. Verify **return route**: asymmetric or missing reverse routes are common.
6. Check ASA interface direction, security level, ACL and NAT.
7. Verify service itself, not just ping (HTTP, DNS or email).
8. Use Packet Tracer Simulation Mode to inspect path and drops.

## Interview Q&A

**What is the purpose of a VLAN?** Logical Layer 2 separation into broadcast domains; prevents broadcasts in VLAN 10 being flooded into VLAN 20. Inter-VLAN communication needs a router or Layer 3 switch.

**Access versus trunk?** An access port normally carries one untagged data VLAN; an 802.1Q trunk can carry several VLAN IDs in tagged frames (native VLAN typically untagged).

**What is a native VLAN?** The VLAN associated with untagged frames on a trunk by default; not necessarily VLAN 1 and not the same concept as the default VLAN.

**Why SVIs?** Give VLANs routed Layer 3 gateway interfaces on a multilayer switch.

**How does a PC in VLAN 10 reach VLAN 20?** Sends to its gateway MAC; L3 switch performs an IP route lookup then sends out VLAN 20 with rewritten Ethernet headers.

**What did OSPF do?** Learned campus and branch networks dynamically; HQ shows OSPF-learned routes and two equal-cost campus next hops.

**What is administrative distance?** Local trust preference between different route sources for the same prefix; OSPF shows 110, static typically 1. **Metric** selects among paths learned by a protocol.

**What is HSRP?** An active/standby first-hop redundancy protocol offering a virtual gateway; must verify actual pair and failover.

**Why STP?** Prevents switching loops when redundant Layer 2 links exist. PortFast accelerates edge-host attachment; BPDU Guard protects edge ports.

**Why DHCP relay?** Client DHCP broadcasts typically don't cross a router; `ip helper-address` on a Layer 3 interface sends requests to the server.

**Why DMZ?** Semi-trusted server segment separated from both private users and ISP-facing outside network.

**Why ASA security levels 100/50/0?** Designate the default relative trust of inside/DMZ/outside; ACL and NAT rules still determine real reachability.

**Does ASA allow all inside-to-DMZ traffic?** Higher-to-lower connections are generally allowed by default, **if** no ACL or other policy prevents it and routing/services work. Never say 'always'.

**Does DMZ-to-inside work by default?** New unsolicited lower-to-higher connections normally require a permit. Replies to inside-initiated permitted sessions can pass through stateful connection tracking.

**What is stateful inspection?** Track permitted flows so related return packets can be matched to session state.

**Why a default route on HQ?** Send traffic for unknown destinations to ASA `10.0.1.1`, while known internal subnets still use specific OSPF/connected routes.

**Why ASA static return routes?** ASA needs a path back to VLAN and branch private subnet destinations via HQ `10.0.1.2`.

**NAT versus PAT?** NAT translates addresses; PAT lets multiple inside flows share one outside address using ports/flow identifiers.

**Which IP will ISP see?** Under successful interface PAT, outside source becomes ASA OUTSIDE IP `203.0.113.2` (test address), not ASA INSIDE `10.0.1.1`.

**Does NAT replace a firewall ACL?** No. NAT rewrites addresses, ACL/policy determines who may initiate which connections.

**How do you make a DMZ server public?** Provide server addressing and routes, configure appropriate static service mapping, narrow inbound OUTSIDE permit, and verify the ISP routes/forwards to the public translated address.

**Can you simply assign `203.0.113.3` to your server?** No: `203.0.113.3` is the **broadcast address** of the ASA's `203.0.113.0/30` link. Another routed public test IP or service PAT is needed.

**What is the project's current security stage?** Interfaces/routing design are documented; PAT and DMZ service publishing require configuration confirmation, access policy and tests. This accurate status is better than claiming the project is fully secured.

## A concise portfolio presentation script

> I designed a Packet Tracer enterprise campus with department VLANs, redundant Layer 3 core connections, dynamic OSPF routing, and two branch WANs. Campus users share infrastructure services such as DHCP and DNS. At the perimeter, I attached an ASA firewall with three security zones: private inside, server DMZ, and simulated ISP outside. The verified HQ routing table shows OSPF reachability to all seven campus subnets and both branch LANs, plus a static default route to ASA. My final validation work is NAT/PAT, controlled DMZ service access, and restrictive ACL tests. The project taught me how switching, IP addressing, routing tables, firewall state and traffic translation interact end-to-end.
