# 04 — Cisco ASA firewall: complete concept and command revision

This is the main **interview revision chapter**. It separates **commands proven by your ASA output** from **commands suggested but not yet proven applied**.

## 1. Firewall and DMZ fundamentals

A **firewall** enforces a network security policy by allowing, rejecting, or dropping traffic based on rules and connection state. A **DMZ** is a separate network for services that may need exposure to untrusted clients (such as web, email or public DNS). Putting a server in a DMZ is not itself sufficient security: routing, NAT, ACLs, OS hardening and service-level restrictions also matter.

An ASA is a **stateful firewall**: it records permitted sessions in its connection table so matching replies are generally allowed without separately permitting each response. New unsolicited inbound traffic is treated differently from replies to existing permitted flows. For TCP it tracks the flow; UDP/ICMP support differs by inspection policy and ASA/Packet Tracer implementation.

### The three actual ASA zones

| Interface | Actual nameif | IP | Trust level | Meaning |
|---|---|---|---|---|
| GigabitEthernet1/1 | `INSIDE` | `10.0.1.1 255.255.255.252` | `100` | Private HQ/campus/branches behind HQ |
| GigabitEthernet1/2 | `DMZ` | `192.168.200.1 255.255.255.0` | `50` | Separate exposed server network |
| GigabitEthernet1/3 | `OUTSIDE` | `203.0.113.2 255.255.255.252` | `0` | Simulated untrusted ISP-facing network |

Higher security number means **higher default trust**. Traditionally traffic initiated from a higher-security interface to a lower-security interface is allowed absent overriding ACL/security policy, while the reverse is not freely allowed. Security-level rules are **defaults**, not a substitute for explicit policy design. An inbound access-group can change the behavior; NAT/routing/inspection must also work. ASA interface names are **case-sensitive in CLI contexts** on some versions; preserve your actual uppercase `INSIDE`, `DMZ`, `OUTSIDE` names.

## 2. Actual interface commands explained

Observed on ASA:

```cisco
interface GigabitEthernet1/1
 nameif INSIDE
 security-level 100
 ip address 10.0.1.1 255.255.255.252
!
interface GigabitEthernet1/2
 nameif DMZ
 security-level 50
 ip address 192.168.200.1 255.255.255.0
!
interface GigabitEthernet1/3
 nameif OUTSIDE
 security-level 0
 ip address 203.0.113.2 255.255.255.252
!
```

| Command | What it literally does | Why in this topology |
|---|---|---|
| `interface GigabitEthernet1/1` | Selects physical interface for configuration | HQ-to-ASA link |
| `nameif INSIDE` | Labels this interface as a security zone, referenced by route/NAT/ACL commands | Separates private users from other traffic |
| `security-level 100` | Assigns maximum traditional ASA interface trust score | Campus is most trusted network (not unlimited access) |
| `ip address 10.0.1.1 255.255.255.252` | Gives inside interface its own /30 address | HQ router `10.0.1.2` can be the next-hop peer |
| `nameif DMZ` | Labels DMZ-facing interface | Isolates semi-trusted public-services segment |
| `security-level 50` | Middle trust score | DMZ has less default trust than the inside |
| `ip address 192.168.200.1 255.255.255.0` | DMZ gateway address | Servers use this address as their default gateway |
| `nameif OUTSIDE` | Labels Internet-facing interface | Used by edge routing, PAT, inbound policy |
| `security-level 0` | Lowest default trust score | ISP/untrusted zone |
| `ip address 203.0.113.2 255.255.255.252` | ASA's test-range outside address | Address selected for interface PAT |
| `!` | Cisco configuration separator (not an executable command) | Readability |

`no shutdown` enables an interface administratively where supported. The observed `show interface ip brief` showed these three interfaces up/up, so their operational interfaces were active at the time.

## 3. Actual inspection and routing commands

```cisco
show interface ip brief
show run interface gigabitEthernet 1/1
show run interface gigabitEthernet 1/2
show run interface gigabitEthernet 1/3
show route
```

- `show interface ip brief`: short physical/logical interface IP and status table, useful first diagnostic.
- `show run interface ...`: filters running config to a particular interface to confirm `nameif`, security level, subnet.
- `show route`: displays connected, static and dynamically learned routes. Your initial ASA output said **Gateway of last resort is not set**: it did not yet have a usable default route.

**Anomaly:** The ASA's `show route` also showed `10.0.0.0/30` as directly connected to INSIDE alongside `10.0.1.0/30`, even though the listed INSIDE interface is `10.0.1.1/30`. That is unexpected for the displayed primary interface configuration; re-check full interface/route output before trusting this entry. It should not be accepted as proof of an additional physical subnet on the ASA.

## 4. Proposed ASA routes (not yet confirmed saved)

```cisco
configure terminal
route INSIDE 10.0.0.0 255.0.0.0 10.0.1.2 1
route INSIDE 172.16.0.0 255.255.0.0 10.0.1.2 1
route OUTSIDE 0.0.0.0 0.0.0.0 203.0.113.1 1
end
```

- `configure terminal`: enter global configuration mode.
- `route INSIDE`: configure a **static route on ASA**, telling the ASA where destinations are reachable, *not* a NAT rule or an ACL.
- `10.0.0.0 255.0.0.0`: summarizes multiple 10.x networks behind HQ. It is broad and can also include nonexistent destinations; in production, prefer precise network summarization.
- `172.16.0.0 255.255.0.0`: a second summary for branch /24 networks.
- `10.0.1.2`: next-hop HQ router. It knows how to reach the actual campus VLANs and branches by OSPF.
- Trailing `1`: **administrative distance** for this static route; it is not a VLAN ID, metric or port number.
- `route OUTSIDE 0.0.0.0 0.0.0.0 ...`: default route for anything without a more specific route.
- `203.0.113.1`: the *assumed* ISP-side next hop; **verify on the ISP before configuring**.
- `end`: return to privileged EXEC mode.

Example: When DMZ server `192.168.200.10` replies to campus PC `10.10.10.10`, ASA needs a route to campus via `10.0.1.2` (in addition to any firewall/NAT requirements). The packet goes DMZ → ASA → HQ → core → PC.

## 5. NAT versus PAT

**NAT** changes an IP source and/or destination address in a packet as configured. **Static NAT** maps a predictable inside address to an outside address; **dynamic NAT** selects a translation when needed. **PAT (Port Address Translation)** lets many connections share an outside address by distinguishing sessions with transport ports or other identifiers.

**Inside local** in this example: `10.10.10.10` (private source). **Translated/outside source**: `203.0.113.2` (ASA's outside IP for outbound PAT). The ASA tracks translations, so return packets can be delivered to the correct original host.

PAT generally does **not** make private hosts reachable for unsolicited inbound sessions. Publishing a DMZ service needs a mapping and an appropriate inbound ACL, plus ISP-side routing.

## 6. Proposed NAT object commands explained

These commands were **given as a plan but have not been confirmed executed**:

```cisco
object network CAMPUS-NAT
 subnet 10.0.0.0 255.0.0.0
 nat (INSIDE,OUTSIDE) dynamic interface

object network BRANCH-NAT
 subnet 172.16.0.0 255.255.0.0
 nat (INSIDE,OUTSIDE) dynamic interface
```

| Command | Explanation | Reason |
|---|---|---|
| `object network CAMPUS-NAT` | Defines a named network object containing the campus source subnet | Easier to read than repeated addresses |
| `subnet 10.0.0.0 255.0.0.0` | Source range matched by this object | Covers all seven `10.*` campus VLAN subnets (but also extra 10/8 addresses) |
| `nat (INSIDE,OUTSIDE)` | Match traffic entering INSIDE and exiting OUTSIDE | Only edge-bound traffic needs this rule, not ordinary INSIDE→DMZ traffic |
| `dynamic` | Create per-flow dynamic mappings | Many clients use translations on demand |
| `interface` | Use the ASA OUTSIDE interface address as translated address | Intended translated source is `203.0.113.2` |
| `object network BRANCH-NAT` | A second source address object | Branch traffic also arrives through HQ on INSIDE |
| `subnet 172.16.0.0 255.255.0.0` | Covers branch networks | Both `172.16.10.0/24` and `172.16.20.0/24` fit inside the summary |

**Illustrative flow**: PC `10.10.10.10:51000` → ISP service. Before translation: source `10.10.10.10`. After PAT: source `203.0.113.2`, with translated port/identifier chosen by ASA. The destination remains the chosen ISP service. Responses translate back only for the tracked flow.

**Important:** Packet Tracer ASA supports a subset of real ASA commands. Check whether `object network` and the specific NAT syntax are supported; syntax and ACL behavior can differ from production ASA versions. If commands are rejected, do not assume PAT was installed.

## 7. ACLs and why they matter

An **ACL** is a set of ordered match/permit/deny rules, often matching source/destination IPs, protocol and port. ASA interface ACLs are commonly applied with `access-group ... in interface ...` to regulate **new connections entering** an interface. An ACL is different from a **route** (where to forward) and from **NAT** (how to rewrite addresses).

For example, `tcp ... eq 80` concerns HTTP on TCP port 80; `tcp ... eq 443` is HTTPS; `udp ... eq 53` and sometimes TCP 53 relate to DNS. `smtp` uses TCP port 25 for server-to-server mail transport. Do not expose services merely because a device exists in the DMZ.

Example conceptual policy:

| Source zone | Destination | Desired access |
|---|---|---|
| INSIDE | DMZ web server | HTTP/HTTPS, perhaps admin services only from authorized management hosts |
| INSIDE | DMZ DNS | DNS as needed |
| OUTSIDE | DMZ web server | Only public TCP 80/443, if intentionally published |
| DMZ | INSIDE | Default deny; exceptions only when justified |
| Guest VLAN | HR/Finance | Deny in an internal ACL, not automatically enforced just by different VLANs |

An ACL with the wrong interface, direction, pre-/post-NAT address, or order can silently drop otherwise correctly routed packets. Real ASA 8.3+ inbound ACLs commonly refer to **real/private destination IPs**, even when static NAT publishes a public address. Packet Tracer behavior must be checked.

## 8. Security levels vs security policies

**Interview trap:** Is INSIDE (100) → DMZ (50) always allowed? Not always. Default security-level rules normally permit higher-to-lower initiation, but an ACL, failed NAT policy, missing routes, unavailable service, or inspection behavior can cause failure. Reverse DMZ-initiated connections to INSIDE are normally denied without permission, while responses to inside-initiated permitted sessions are statefully tracked.

**Another trap:** Does a firewall need OSPF? No. ASA can forward using connected and static routes, as in our proposed design. Firewalls *can* run dynamic routing on supported platforms, but this lab need not.

## 9. ASA verification and troubleshooting commands

```cisco
show interface ip brief
show run
show route
show nat
show xlate
show conn
show access-list
```

- `show nat`: configured NAT rules, often including hit information depending on version.
- `show xlate`: active translation table (if implemented by Packet Tracer model).
- `show conn`: active tracked sessions.
- `show access-list`: ACL entries/hit counts when supported.
- `show run`: check actual active configurations; do not confuse proposed command snippets with present configuration.

Real ASA devices may also support `packet-tracer input ...` for simulated packet analysis; Cisco **Packet Tracer simulator** may omit the ASA command or implement it incompletely.

## 10. Layer-by-layer troubleshooting

1. **Physical:** ports and link state up/up?
2. **Host:** valid IP, subnet mask, default gateway, service listening?
3. **Routing:** HQ default, ASA back routes, ASA ISP default, ISP return path?
4. **Translation:** correct NAT interface pair and source subnet? existing `show xlate`?
5. **Policy:** any ACL block, interface security levels, service port mismatch?
6. **Application:** DNS A record pointing to valid IP, web service enabled, SMTP server configured?
7. **Simulation:** use Packet Tracer's Simulation mode to inspect drops and ensure the feature is supported.

## 11. One-minute interview explanation

> I built a three-zone ASA edge firewall. The INSIDE interface at `10.0.1.1/30`, level 100, connects to our HQ router; the DMZ interface at `192.168.200.1/24`, level 50, isolates externally oriented servers; and the OUTSIDE interface at `203.0.113.2/30`, level 0, points to a simulated ISP. HQ has a default route toward ASA, and ASA needs reverse routes to campus and branches. I designed PAT so campus and branch clients can share ASA's outside address. The remaining task is to verify translation, test DMZ connectivity, and apply static publishing plus restrictive ACLs. Security levels define defaults, but safe service exposure requires explicit policies and verification.
