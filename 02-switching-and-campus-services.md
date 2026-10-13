# 02 — Switching, VLANs and campus services

**Note:** The commands in this file are explanatory IOS examples, **not exports** of exact running configurations. Adjust interface numbers, gateway addresses, VLAN memberships and device features after inspection.

## VLAN fundamentals

A VLAN logically divides a physical switch into separate Layer 2 broadcast domains. A switch learns **source MAC addresses** on each VLAN/port and forwards known unicast frames to the learned destination port; unknown unicasts and ARP broadcasts are flooded **within that VLAN**, not across all VLANs. To send IP packets between VLANs, a Layer 3 gateway is needed.

```cisco
vlan 10
 name IT
vlan 20
 name SALES
interface fastEthernet0/1
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 spanning-tree bpduguard enable
```

- `vlan 10`: creates VLAN 10 in the local switch database.
- `name IT`: useful management label; it does not itself change traffic forwarding.
- `interface fastEthernet0/1`: choose a physical switch interface.
- `switchport mode access`: force a normal access port instead of negotiating trunk operation.
- `switchport access vlan 10`: untagged data frames on that port belong to VLAN 10.
- `spanning-tree portfast`: lets a known end-host port move to forwarding quickly; do not use casually on switch-to-switch links.
- `spanning-tree bpduguard enable`: err-disables a PortFast edge port if BPDUs are received, protecting against accidental switches/loops.

## Trunks and 802.1Q

```cisco
interface gigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,50,60,70
```

A trunk carries multiple VLANs using **802.1Q tags**. The native VLAN's frames are normally untagged, unless native tagging is explicitly enabled and supported. **Default VLAN 1** is the initial VLAN for switch ports; **native VLAN** is a trunk behavior, and can be changed independently. Both trunk endpoints must agree on the native VLAN and permitted VLANs. DTP can dynamically negotiate trunks on compatible Cisco switches, but hard-coded trunk/access modes are easier to audit. `show interfaces trunk` confirms operational trunking.

For an IP phone, an access interface can carry a data VLAN and a separate tagged voice VLAN (platform permitting):

```cisco
interface fastEthernet0/10
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 80
```

The voice VLAN is an **illustrative example**, not a confirmed configured VLAN 80 in this project.

## SVIs and inter-VLAN routing

```cisco
ip routing
interface vlan 10
 ip address 10.10.10.1 255.255.255.0
 no shutdown
```

- `ip routing`: enables IPv4 packet forwarding on a multilayer switch.
- `interface vlan 10`: creates/enters a switched virtual interface, a Layer 3 interface representing VLAN 10.
- `ip address ...`: provides a potential default gateway for hosts in the subnet.
- `no shutdown`: administratively enables the SVI; line protocol also depends on VLAN/port state.

**Caution:** `10.10.10.1` is *illustrative*, not confirmed as the final gateway. If HSRP is used, hosts typically use the **virtual IP**, not a physical SVI IP.

### Inter-VLAN packet walkthrough

PC-A (`10.10.10.10`) sends to PC-B (`10.20.10.20`). Because the destination is outside its /24, PC-A ARPs for its default gateway, sends an Ethernet frame to the gateway MAC, and the multilayer switch performs an IP route lookup. The switch then ARPs for (or uses the known MAC of) PC-B and forwards a new frame in VLAN 20. IP source/destination normally remain unchanged; the frame's MAC addresses change across the routed hop.

## STP / PVST

Redundant physical switch links can create loops that amplify broadcasts and cause MAC flapping. Spanning Tree elects a root bridge and places selected redundant ports into non-forwarding states. With PVST, Cisco can maintain a spanning-tree instance per VLAN. Root selection uses bridge priority plus MAC; lower bridge ID wins. Verify with `show spanning-tree vlan 10` and note root and port roles.

Illustrative root placement:

```cisco
spanning-tree vlan 10,20,30 root primary
```

This chooses the switch as a preferred root for selected VLANs; it is **not** proof a particular core is root until tested.

## HSRP

First-hop redundancy uses two eligible Layer 3 gateways with individual real IPs and a shared virtual IP/MAC. One is **active**, another **standby**. Clients use the virtual IP as their default gateway. HSRP does not automatically make a single router/firewall edge redundant.

```cisco
interface vlan 10
 standby 10 ip 10.10.10.254
 standby 10 priority 110
 standby 10 preempt
```

- `standby 10`: HSRP group 10 on the interface.
- `ip ...`: virtual gateway (example only; must be unused in the correct subnet).
- `priority 110`: prefers this device over the default priority 100.
- `preempt`: lets the higher-priority router retake the active role after recovery.

Verify `show standby brief` on both devices, with distinct real SVI IPs and matching virtual IP/group. The lab's exact working HSRP addressing and failover were not independently verified.

## DHCP and relay

DHCP dynamically gives clients IP addresses, masks, gateways and optionally DNS server addresses. Initial handshake: **DORA** (Discover, Offer, Request, Acknowledge). Because DHCP broadcasts do not normally cross routers, a Layer 3 VLAN interface can relay them:

```cisco
interface vlan 10
 ip helper-address <ACTUAL_DHCP_SERVER_IP>
```

`ip helper-address` forwards certain UDP broadcasts to the remote server; use the real DHCP server address (not supplied in the verified outputs). Check that server pools match each VLAN and use the actual default gateway/virtual IP.

## DNS

DNS maps names to IP addresses (such as A and AAAA records). The internal DNS service is separate from the **public DNS server in the DMZ** in the screenshot. Clients need a valid configured resolver and the resolver must have a relevant record for a successful name-based test. A ping by IP is not a DNS test.

## Helpful commands

```cisco
show vlan brief
show interfaces trunk
show mac address-table
show spanning-tree
show spanning-tree vlan 10
show standby brief
show ip interface brief
show ip route
show running-config
```

When documenting results, capture both the command output and an end-device connectivity test.
