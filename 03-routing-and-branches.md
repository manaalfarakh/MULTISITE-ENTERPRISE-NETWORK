# 03 — Routing, OSPF and branches

## What routing means here

A switch routes between local VLANs using its SVIs. The HQ router routes among campus next hops, both remote branches, and the firewall. Each Layer 3 hop consults its routing table. Longest-prefix match is checked first; if multiple routes have the same prefix length, route selection then considers administrative distance and protocol metrics.

## Actual HQ routing-table evidence

The HQ output showed:

```text
C  10.0.0.0/30  directly connected, GigabitEthernet0/1
L  10.0.0.2/32  directly connected, GigabitEthernet0/1
C  10.0.0.4/30  directly connected, GigabitEthernet0/0
L  10.0.0.6/32  directly connected, GigabitEthernet0/0
C  10.0.1.0/30  directly connected, GigabitEthernet0/2
L  10.0.1.2/32  directly connected, GigabitEthernet0/2
O  10.10.10.0/24 [110/2] via 10.0.0.5 and via 10.0.0.1
O  10.20.10.0/24 [110/2] via 10.0.0.5 and via 10.0.0.1
O  10.30.10.0/24 [110/2] via 10.0.0.5 and via 10.0.0.1
O  10.40.10.0/24 [110/2] via 10.0.0.5 and via 10.0.0.1
O  10.50.10.0/24 [110/2] via 10.0.0.5 and via 10.0.0.1
O  10.60.10.0/24 [110/2] via 10.0.0.5 and via 10.0.0.1
O  10.70.10.0/24 [110/2] via 10.0.0.5 and via 10.0.0.1
C  172.16.1.0/30 directly connected, Serial0/3/0
L  172.16.1.2/32 directly connected, Serial0/3/0
C  172.16.2.0/30 directly connected, Serial0/3/1
L  172.16.2.2/32 directly connected, Serial0/3/1
O  172.16.10.0/24 [110/65] via 172.16.1.1, Serial0/3/0
O  172.16.20.0/24 [110/65] via 172.16.2.1, Serial0/3/1
S* 0.0.0.0/0 [1/0] via 10.0.1.1
```

`C` connected; `L` local host address; `O` OSPF intra-area; `S*` static candidate default. The campus routes show **two equal-cost next hops**, demonstrating ECMP route installation (not necessarily simultaneous load usage in any given packet flow). `[110/2]` means OSPF administrative distance 110 and metric 2; `[110/65]` uses metric 65. The `S*` route makes ASA the default next hop for destinations HQ does not know specifically.

## Why OSPF?

OSPF is a link-state interior gateway protocol. Routers advertise link-state information, build a link-state database for an area, and run shortest-path calculation. Area 0 is the backbone; use the exact configured area from your routers rather than assuming. OSPF uses IP protocol 89 (not TCP/UDP port 89), and usually discovers neighbors with multicast `224.0.0.5` on broadcast networks. Neighbor formation and routes require consistent interface/subnet/network-type and area settings (plus compatible options).

Illustrative router configuration:

```cisco
router ospf 1
 network 172.16.1.0 0.0.0.3 area 0
 network 172.16.10.0 0.0.0.255 area 0
```

- `router ospf 1` starts OSPF process 1, locally significant, not an autonomous system number.
- `network ... 0.0.0.3` matches interfaces belonging to the /30 network. The wildcard mask is inverse to the subnet mask.
- `area 0` places matching interfaces in OSPF area 0.

An exact configuration on HQ needs the full running config; the sample is explanatory and might actually belong on a branch depending on link addressing.

## Why HQ has a default route

```cisco
ip route 0.0.0.0 0.0.0.0 10.0.1.1
```

If no more specific route exists, send the packet to ASA's inside address. Without this route, HQ initially reported **Gateway of last resort is not set**. After the command, its route table showed `S* ... via 10.0.1.1`.

## Example campus-to-branch packet path

IT PC → its VLAN default gateway/SVI → core switch routing table → HQ router → serial Branch 1 WAN → branch router → branch access switch → target host. Responses follow routing independently; asymmetric paths can exist when more than one valid route is present.

## Checking OSPF

```cisco
show ip ospf neighbor
show ip route ospf
show ip protocols
show ip interface brief
traceroute 172.16.10.10
```

Use an **actual host IP** for testing; `.10` above is only an example. A visible route does not guarantee host reachability; check host gateways, ACLs and physical interfaces.
