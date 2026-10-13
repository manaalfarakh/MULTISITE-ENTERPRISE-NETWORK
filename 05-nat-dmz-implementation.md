# 05 — Completing NAT/PAT and DMZ connectivity (not yet verified)

> **Status: work in progress.** These are careful implementation instructions, **not claims of commands already applied**. Save/export existing configurations before edits. Check platform syntax because Packet Tracer ASA has a limited CLI.

## Stage A: Confirm your baseline

### On HQ router

```cisco
show ip route
show ip interface brief
ping 10.0.1.1
```

HQ's routing table **already showed** default `S* 0.0.0.0/0 via 10.0.1.1`, connected `10.0.1.0/30` and OSPF routes to VLANs and branches. Do not add a duplicate default route if present.

### On ASA

```cisco
show interface ip brief
show route
show run
show nat
```

Confirmed interface/IP assignments are INSIDE `.1/30` on `10.0.1.0/30`, DMZ `.1/24` on `192.168.200.0/24`, OUTSIDE `.2/30` on `203.0.113.0/30`. One **unexpected ASA connected route to `10.0.0.0/30`** appeared; inspect this before making changes.

### On ISP router

```cisco
show ip interface brief
show ip route
```

Check whether the ASA-facing IP actually is `203.0.113.1/30`. The next-hop must exist before an ASA default route is useful.

## Stage B: Routes

If ISP really is `203.0.113.1`, add missing ASA routes:

```cisco
conf t
route INSIDE 10.0.0.0 255.0.0.0 10.0.1.2 1
route INSIDE 172.16.0.0 255.255.0.0 10.0.1.2 1
route OUTSIDE 0.0.0.0 0.0.0.0 203.0.113.1 1
end
show route
```

Prefer exact inside prefixes or a well-justified safe summary in production; these ranges are intentionally broad for lab convenience. If the ISP is doing routing rather than just hosting a single test endpoint, ensure it knows how to reach any simulated downstream test subnet. PAT return traffic generally goes to ASA's outside address, which sits directly on the ISP link.

## Stage C: Configure outbound PAT

Only after checking `show nat` for existing rules, configure **if supported**:

```cisco
conf t
object network CAMPUS-NAT
 subnet 10.0.0.0 255.0.0.0
 nat (INSIDE,OUTSIDE) dynamic interface
object network BRANCH-NAT
 subnet 172.16.0.0 255.255.0.0
 nat (INSIDE,OUTSIDE) dynamic interface
end
show nat
```

Why two objects? One matches campus source networks, the other matches branch source networks. Both arrive at ASA on **INSIDE** (through HQ). ASA replaces original source with its `OUTSIDE` interface IP `203.0.113.2` for ISP-bound connections.

**Test:** From a campus host, ping the verified ISP interface. Also test from one PC in each branch. Confirm routing first; then review `show nat`, `show xlate`, `show conn` as available. ICMP can behave differently from TCP depending on inspection settings. An ISP-interface ping is **not proof of real public Internet**.

## Stage D: Internal access to DMZ

1. On **each DMZ server**, check IP in `192.168.200.0/24`, mask `255.255.255.0`, default gateway **`192.168.200.1`**, and enabled web/mail/DNS services as appropriate. Individual IPs were not provided; **do not assign conflicting addresses**.
2. From a DMZ server, test the ASA DMZ interface `192.168.200.1`.
3. On ASA, inspect routes to campus `10.10.10.0/24` etc. through HQ `10.0.1.2`.
4. On HQ/core, make sure campus hosts use their correct SVI/HSRP default gateway; HQ's route table already shows connected/OSPF routing.
5. From campus PC use the **actual DMZ server IP**: test HTTP with Packet Tracer Web Browser to `http://<actual-dmz-web-ip>`, mail via configured mail client, and DNS with a domain lookup if supported.
6. INSIDE→DMZ flows may be allowed under default security-level policy, but an ACL may restrict them. Review `show access-list` and `show run` if traffic fails.
7. **No INSIDE→OUTSIDE PAT rule should be needed just for INSIDE→DMZ access.** Using the wrong NAT interface pair can break access.

## Stage E: Public DMZ web server (future work)

First choose: actual private web server IP, an available external/public address (or a port-forward strategy with ASA's interface IP), and whether HTTP/HTTPS is required. The current `/30` link has only two host addresses (`203.0.113.1` ISP and `.2` ASA), so **there is no third free address inside that /30 for a conventional separate one-to-one mapped public server**. To do standalone one-to-one public static NAT you would need an extra routed/test public IP from the simulated ISP; alternatively use interface PAT / static service port translation **if supported by this Packet Tracer ASA model**.

Illustrative real-ASA object NAT syntax for a separately routed public test address:

```cisco
! EXAMPLE ONLY — DON'T PASTE WITH THESE PLACEHOLDERS
object network DMZ-WEB
 host <REAL_DMZ_WEB_SERVER_IP>
 nat (DMZ,OUTSIDE) static <EXTRA_ROUTED_PUBLIC_TEST_IP>
```

Then apply an **inbound OUTSIDE ACL** allowing only intended TCP destination services (e.g., 80/443) to the appropriate **real DMZ server IP on ASA 8.3+**. Exact ACL syntax depends on installed ASA version and Packet Tracer feature support; do not enable `permit ip any any` on OUTSIDE. Keep DMZ→INSIDE initiation restricted. The ISP must be able to deliver traffic to the translated public IP, not merely the private server address.

## Failure diagnosis table

| Symptom | Check |
|---|---|
| Campus PC cannot ping ASA INSIDE | Core↔HQ connectivity, default gateway, HQ-to-ASA interface, filters |
| ASA can reach HQ but not VLAN PC | ASA return routes via `10.0.1.2`, core routing, host gateway |
| Campus PC reaches DMZ gateway but not server | Server IP/mask/gateway, switch link/VLAN, firewall ACL, host service |
| DMZ server reaches gateway but not campus | ASA routes, DMZ→INSIDE initiation policy (usually blocked), test direction |
| PCs cannot reach ISP | HQ default route, ASA default route, NAT object rule, ISP link/endpoint |
| NAT exists but fails | Source object match, interface pair (`INSIDE`,`OUTSIDE`), ASA feature support, ACL/inspection |
| HTTP fails while ping works | HTTP service state, browser URL/port, ACL for TCP 80, server policy |
| Domain names fail but IP addresses work | DNS resolver address, DNS server service and records, UDP/TCP 53 reachability |

## Completion evidence to commit

Screenshots or text logs: ASA `show run`, `show route`, `show nat`, `show xlate` (if supported); HQ `show ip route`; core VLAN/trunks/HSRP; browser page from internal PC; ISP-side HTTP to public mapping after static NAT; negative tests demonstrating unintended access blocked. Avoid exposing real secrets in exported configs.
