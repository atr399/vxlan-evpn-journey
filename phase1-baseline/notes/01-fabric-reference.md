# Phase 1: VXLAN-EVPN Fabric — Complete Reference Guide

This document is the full reference for Phase 1. Each configuration command is explained with the WHY. Each verification command shows what good output looks like, what bad output looks like, and what to investigate when wrong.

Reading order: skim section 1 first to set context, then jump to any section as needed.

---

## Table of contents

1. Mental model
2. Topology and IP plan
3. Configuration: SPINES (segment by segment)
4. Configuration: LEAVES (segment by segment)
5. Configuration: HOSTS
6. Verification: underlay (OSPF)
7. Verification: overlay (BGP-EVPN)
8. Verification: VXLAN data plane
9. Verification: tenant L2 and L3
10. Verification: end-to-end traffic
11. Packet flow walkthrough
12. Common mistakes and their symptoms
13. Feature dependency cheat sheet

---

## 1. Mental model

VXLAN-EVPN is **two logical networks running on the same physical hardware**:

### Underlay
- Plain IP network. OSPF area 0 with /30 point-to-point links.
- Carries IP packets only. Knows nothing about tenants, MACs, VLANs, VXLAN.
- Job: get an IP packet from leaf1's loopback to leaf2's loopback.

### Overlay
- Tenant Ethernet frames encapsulated in VXLAN (UDP port 4789).
- Tunnels run leaf-to-leaf, never touch spines as endpoints.
- Control plane: BGP with the L2VPN EVPN address family.

### Why spines and leaves do different things

**Spines** = transit + BGP route reflectors. They route IP packets in the underlay and reflect EVPN routes between leaves. They never encapsulate VXLAN, never have tenant VRFs, never have SVIs.

**Leaves** = VTEPs (VXLAN Tunnel Endpoints). Where hosts physically connect. Leaves encapsulate frames into VXLAN going out, decapsulate VXLAN coming in, have tenant VRFs and SVIs, run anycast gateway.

### Why route reflectors

iBGP rule: a router does NOT re-advertise a route learned from one iBGP peer to another iBGP peer. Without route reflection, leaf1's EVPN routes would arrive at the spine and stop there. Route reflector configuration explicitly breaks this rule for its clients (the leaves), so EVPN routes flow leaf → spine → other leaves.

### Why ingress replication

VXLAN needs to handle BUM traffic (Broadcast, Unknown unicast, Multicast — for example ARP). Two methods:

- **Multicast underlay**: every leaf joins a multicast group; BUM is sent once and replicated by the underlay. Requires PIM. More efficient at large scale.
- **Ingress replication (what we use)**: sending leaf creates one unicast copy per remote VTEP. No multicast in the underlay. Simpler. BGP-EVPN Type-3 routes advertise the list of VTEPs to replicate to.

For labs and small/medium fabrics, ingress replication is the standard choice.

---

## 2. Topology and IP plan
              spine1                       spine2
          lo0: 10.0.0.1/32             lo0: 10.0.0.2/32
          AS 65000 (RR)                AS 65000 (RR)
            /      \                     /      \
  10.1.1.0/30   10.1.2.0/30   10.1.3.0/30   10.1.4.0/30
          /          \                /          \
     leaf1                                    leaf2
   lo0: 10.0.0.11/32                        lo0: 10.0.0.12/32
   lo1: 10.0.1.11/32 (VTEP)                 lo1: 10.0.1.12/32 (VTEP)
         |                                       |
      Eth1/9                                  Eth1/9
         |                                       |
      host1                                   host2
   10.10.10.10/24                          10.10.10.20/24
VLAN 10 (host VLAN)  -> L2VNI 10010
VRF tenant1          -> L3VNI 50001
VLAN 100             -> L3VNI transit VLAN (carries the L3VNI)
Anycast gateway: 10.10.10.1 / 0000.2222.3333
OSPF area 0.0.0.0 across the whole underlay
BGP AS 65000 (iBGP), spines as RRs

### Loopback purposes

| Interface | Purpose |
|---|---|
| `loopback0` (lo0) on all nodes | Router-id, BGP source, control-plane identity |
| `loopback1` (lo1) on leaves only | VTEP source — the IP other leaves tunnel to |

Two separate loopbacks because in vPC scenarios (Phase 3), lo1 takes a shared secondary IP. Keeping it separate from the unique lo0 router-id is the standard practice.

---

## 3. Configuration: SPINES (segment by segment)

### Segment 3.1 — Global features
feature ospf
feature bgp
nv overlay evpn

| Command | Why |
|---|---|
| `feature ospf` | Enables OSPF for the underlay. Without this, OSPF process won't even configure. |
| `feature bgp` | Enables BGP for the overlay control plane. |
| `nv overlay evpn` | **Critical** — enables the L2VPN EVPN address family in BGP. Without this, `address-family l2vpn evpn` under `router bgp` won't accept. This is not a `feature` command (it's a separate global knob), so it's easy to forget. |

Note: spines do NOT need `feature nv overlay`, `feature interface-vlan`, or `feature vn-segment-vlan-based` because they don't encapsulate VXLAN or hold tenant config.

### Segment 3.2 — Loopback0 (router-id)
interface loopback0
ip address 10.0.0.1/32
ip router ospf 1 area 0.0.0.0

| Command | Why |
|---|---|
| `ip address 10.0.0.1/32` | The spine's identity. Used as router-id for OSPF and BGP. Always /32 — a host route, not a subnet. |
| `ip router ospf 1 area 0.0.0.0` | Advertise this loopback into OSPF so every leaf has a route to it. Needed for BGP sessions sourced from loopback0. |

### Segment 3.3 — P2P links to leaves
interface Ethernet1/1
description to-leaf1
no switchport
ip address 10.1.1.1/30
ip router ospf 1 area 0.0.0.0
no shutdown

| Command | Why |
|---|---|
| `no switchport` | NX-OS interfaces default to L2 switchport. We need L3 routing on the underlay, so flip to L3. |
| `ip address 10.1.1.1/30` | /30 gives exactly 2 usable IPs — one for each side of the p2p link. No waste. |
| `ip router ospf 1 area 0.0.0.0` | Run OSPF on this link. Without it, OSPF neighborship doesn't form. |
| `no shutdown` | NX-OS interfaces default admin-down. Always required. |

### Segment 3.4 — OSPF process
router ospf 1
router-id 10.0.0.1

| Command | Why |
|---|---|
| `router ospf 1` | `1` is the process ID — locally significant, doesn't have to match other devices. |
| `router-id 10.0.0.1` | Explicit router-id. Should match loopback0 for clarity. If not set, OSPF picks the highest loopback IP, which is usually fine but explicit is safer. |

### Segment 3.5 — BGP basics and EVPN AF
router bgp 65000
router-id 10.0.0.1
address-family l2vpn evpn

| Command | Why |
|---|---|
| `router bgp 65000` | Single AS for the whole fabric (iBGP design). |
| `router-id 10.0.0.1` | BGP needs a stable router-id; match the loopback. |
| `address-family l2vpn evpn` | Globally enable the EVPN AF. Required before any neighbor-level EVPN config will work. Depends on `nv overlay evpn`. |

### Segment 3.6 — Peer template (RR)
template peer LEAF-RR-CLIENT
remote-as 65000
update-source loopback0
address-family l2vpn evpn
send-community
send-community extended
route-reflector-client

| Command | Why |
|---|---|
| `template peer LEAF-RR-CLIENT` | A reusable bundle of settings. Apply once to N neighbors. Avoids copy/paste errors. |
| `remote-as 65000` | Same AS = iBGP. |
| `update-source loopback0` | Source BGP TCP from loopback0. Loopback is always-up; physical interfaces flap. |
| `send-community` | Send standard BGP communities. |
| `send-community extended` | Send extended communities. **EVPN route targets live here** — without this, RTs aren't transmitted and import/export breaks. |
| `route-reflector-client` | Tell BGP "this neighbor is my RR client" — I will re-advertise its routes to other iBGP peers. This is what makes the spine a route reflector. |

### Segment 3.7 — Apply template to leaves
neighbor 10.0.0.11 inherit peer LEAF-RR-CLIENT
neighbor 10.0.0.12 inherit peer LEAF-RR-CLIENT

Inherits all template settings. Cleaner than typing the same block twice.

---

## 4. Configuration: LEAVES (segment by segment)

### Segment 4.1 — Global features
feature ospf
feature bgp
feature interface-vlan
feature vn-segment-vlan-based
feature nv overlay
nv overlay evpn

| Command | Why |
|---|---|
| `feature interface-vlan` | Allows creating SVIs (`interface Vlan10`). Without it, SVI commands are rejected. |
| `feature vn-segment-vlan-based` | Allows mapping a VLAN to a VXLAN VNI (`vn-segment 10010` under VLAN). |
| `feature nv overlay` | Enables the VXLAN encapsulation engine. Required to create `interface nve1`. |
| `nv overlay evpn` | Same as on spines — enables EVPN AF in BGP. |

### Segment 4.2 — Loopback0 (router-id)

Same as spines. Identity loopback for BGP and OSPF.

### Segment 4.3 — Loopback1 (VTEP source)
interface loopback1
description VTEP-source
ip address 10.0.1.11/32
ip router ospf 1 area 0.0.0.0

| Command | Why |
|---|---|
| `loopback1` | Dedicated VTEP loopback, separate from lo0. |
| `ip address 10.0.1.11/32` | This is the IP other leaves use as the VXLAN tunnel destination. |
| `ip router ospf 1 area 0.0.0.0` | Must be reachable in the underlay or remote leaves can't tunnel to us. |

### Segment 4.4 — P2P links to spines

Same pattern as spines. `no switchport`, p2p /30, OSPF.

### Segment 4.5 — OSPF process

Same as spines but with the leaf's router-id.

### Segment 4.6 — BGP and EVPN AF
router bgp 65000
router-id 10.0.0.11
address-family l2vpn evpn
template peer SPINE-RR
remote-as 65000
update-source loopback0
address-family l2vpn evpn
send-community
send-community extended
neighbor 10.0.0.1 inherit peer SPINE-RR
neighbor 10.0.0.2 inherit peer SPINE-RR

Note: leaves do NOT have `route-reflector-client`. They are RR clients OF the spines, not RRs themselves.

### Segment 4.7 — VRF tenant1 and L3VNI
vlan 100
name L3VNI-tenant1
vn-segment 50001
vrf context tenant1
vni 50001
rd auto
address-family ipv4 unicast
route-target both auto
route-target both auto evpn
interface Vlan100
description L3VNI-tenant1
no shutdown
mtu 9216
vrf member tenant1
ip forward
no ip redirects

| Command | Why |
|---|---|
| `vlan 100` + `vn-segment 50001` | NX-OS implementation detail — the L3VNI rides on a "transit VLAN" (VLAN 100). The VLAN itself never has hosts; it's purely the L3VNI's plumbing. |
| `vrf context tenant1` | Create the tenant VRF. Separates tenant1's IP routing from other tenants and from default VRF. |
| `vni 50001` | Bind the L3VNI value to this VRF. Used as VNI in VXLAN header when traffic is routed across the fabric. |
| `rd auto` | Auto-generate the route distinguisher. Format becomes `<router-id>:<auto-id>` — uniquely identifies this VRF's advertisements. |
| `route-target both auto` | Standard VPN route-target for import and export (legacy MPLS-style). |
| `route-target both auto evpn` | EVPN-specific route-target. Tells other leaves which VRF to import these routes into. |
| `interface Vlan100` + `vrf member tenant1` + `ip forward` | The L3VNI SVI. No IP address — it's a forwarding-only conduit between the VRF and the NVE interface. `mtu 9216` for jumbo to accommodate VXLAN overhead. |

### Segment 4.8 — L2VNI (host VLAN bridging)
vlan 10
name HOST_VLAN
vn-segment 10010
evpn
vni 10010 l2
rd auto
route-target import auto
route-target export auto

| Command | Why |
|---|---|
| `vlan 10` + `vn-segment 10010` | VLAN 10 on the leaf is mapped to VXLAN VNI 10010 across the fabric. Any leaf with VLAN 10 + VNI 10010 mapping participates in this L2 segment. |
| `evpn` / `vni 10010 l2` | Declare this VNI as an L2 EVPN instance (a MAC-VRF). |
| `rd auto`, `route-target import/export auto` | Same idea as VRF — uniquely identify and propagate MAC-VRF routes. |

### Segment 4.9 — NVE interface
interface nve1
description VXLAN-VTEP
no shutdown
host-reachability protocol bgp
source-interface loopback1
member vni 10010
ingress-replication protocol bgp
member vni 50001 associate-vrf

| Command | Why |
|---|---|
| `interface nve1` | The VXLAN tunnel interface. Each leaf has one. |
| `host-reachability protocol bgp` | Use BGP-EVPN to learn remote MACs, not legacy flood-and-learn. **Without this, the fabric falls back to data-plane learning** — broken EVPN. |
| `source-interface loopback1` | Use lo1 as the source IP of all VXLAN packets. Other leaves tunnel back to this IP. |
| `member vni 10010` | Attach L2VNI 10010 to this NVE. |
| `ingress-replication protocol bgp` | Use ingress replication for BUM, with VTEP discovery via BGP-EVPN Type-3 routes. |
| `member vni 50001 associate-vrf` | Attach the L3VNI and associate it with the VRF. Routed traffic between subnets in tenant1 uses this VNI. |

### Segment 4.10 — Anycast gateway
fabric forwarding anycast-gateway-mac 0000.2222.3333
interface Vlan10
description anycast-gw-VLAN10
no shutdown
mtu 9216
vrf member tenant1
no ip redirects
ip address 10.10.10.1/24
fabric forwarding mode anycast-gateway

| Command | Why |
|---|---|
| `fabric forwarding anycast-gateway-mac 0000.2222.3333` | Define the shared MAC used by every leaf for its anycast gateway SVIs. **Same MAC on all leaves** — that's the "anycast" part. Hosts always see this MAC for their gateway, regardless of which leaf they're connected to. |
| `interface Vlan10` | The L3 SVI for host VLAN 10. |
| `vrf member tenant1` | Put this SVI in the tenant VRF. |
| `no ip redirects` | Required for anycast gateway. ICMP redirects would confuse hosts about which leaf is "the" gateway. |
| `ip address 10.10.10.1/24` | The gateway IP. **Same IP on all leaves** — also part of "anycast". |
| `fabric forwarding mode anycast-gateway` | Activate anycast-gateway behavior on this SVI. Without this, NX-OS would treat it as a normal SVI and you'd have IP/MAC duplication conflicts. |

### Segment 4.11 — Host-facing port
interface Ethernet1/9
description to-host
switchport
switchport mode access
switchport access vlan 10
no shutdown

| Command | Why |
|---|---|
| `switchport` | Reverse the default `no switchport` we put on underlay links. Host port is L2. |
| `switchport mode access` | Host doesn't tag — access port handles untagged frames. |
| `switchport access vlan 10` | Put host traffic into VLAN 10, which is mapped to L2VNI 10010. |

---

## 5. Configuration: HOSTS

Inside each host container:
ip addr add 10.10.10.10/24 dev eth1
ip link set eth1 up
ip route add default via 10.10.10.1

Hosts only need IP + default gateway. Anycast gateway IP `10.10.10.1` lives on whichever leaf they're connected to.

---

## 6. Verification: underlay (OSPF)

### show ip ospf neighbors
leaf1# show ip ospf neighbors
Neighbor ID     Pri State            Up Time  Address         Interface
10.0.0.1          1 FULL/DR          00:15:32 10.1.1.1        Eth1/1
10.0.0.2          1 FULL/DR          00:15:28 10.1.3.1        Eth1/2

**What to look for:**
- `State: FULL` — OSPF adjacency is fully established.
- Should see one neighbor per underlay link (each leaf sees both spines).
- `Up Time` increasing on subsequent checks = stable.

**Bad output and what it means:**
- `INIT` or `EXSTART` stuck — interface up but OSPF not converging. Check `ip router ospf 1 area 0.0.0.0` is on the interface and OSPF is running.
- `2WAY` — neighborship formed but not exchanging routes. Usually a configuration mismatch (area number, MTU).
- Missing neighbor entirely — physical link down, OSPF not enabled on interface, or IP not configured.

### show ip route ospf-1
leaf1# show ip route ospf-1
10.0.0.1/32, ubest/mbest: 2/0
*via 10.1.1.1, Eth1/1, [110/41], 00:14:55, ospf-1, intra
*via 10.1.3.1, Eth1/2, [110/41], 00:14:55, ospf-1, intra
10.0.0.2/32, ubest/mbest: 1/0
*via 10.1.3.1, Eth1/2, [110/41], 00:14:55, ospf-1, intra
10.0.0.12/32, ubest/mbest: 2/0
*via 10.1.1.1, Eth1/1, [110/81], 00:14:55, ospf-1, intra
*via 10.1.3.1, Eth1/2, [110/81], 00:14:55, ospf-1, intra
10.0.1.12/32, ubest/mbest: 2/0
*via 10.1.1.1, Eth1/1, [110/81], 00:14:55, ospf-1, intra
*via 10.1.3.1, Eth1/2, [110/81], 00:14:55, ospf-1, intra

**What to look for:**
- Routes to all loopbacks of other devices (lo0 of spines and other leaves, lo1 of other leaves).
- **`ubest 2/0` = ECMP working** — there are 2 paths (via both spines) and underlay traffic is load-balanced. This is the whole point of spine-leaf.
- Cost `41` = directly connected loopback path. Cost `81` = via remote leaf (transits a spine).

**Bad output:**
- Missing routes to remote leaf loopbacks = remote leaf's OSPF is broken or loopback isn't advertised. Underlay is broken; VXLAN can't work.
- Only 1 path when there should be 2 = one spine link is down or OSPF isn't running on it. Fabric works but no redundancy.

### Underlay ping test
leaf1# ping 10.0.1.12 source 10.0.1.11
PING 10.0.1.12 (10.0.1.12): 56 data bytes
64 bytes from 10.0.1.12: icmp_seq=0 ttl=253 time=2.345 ms

**Why this specific ping:** between VTEP loopbacks. If this works, the VXLAN tunnel has a working IP path. If it fails, no point looking at BGP or NVE — fix the underlay first.

---

## 7. Verification: overlay (BGP-EVPN)

### show bgp l2vpn evpn summary
leaf1# show bgp l2vpn evpn summary
BGP router identifier 10.0.0.11, local AS number 65000
BGP table version is 15, L2VPN EVPN config peers 2, capable peers 2
7 network entries and 9 paths using 2548 bytes of memory
Neighbor   V   AS    MsgRcvd  MsgSent  TblVer  InQ  OutQ  Up/Down  State/PfxRcd
10.0.0.1   4   65000     8        11      15    0     0  00:01:04 2
10.0.0.2   4   65000     8        11      15    0     0  00:00:52 2

**What to look for:**
- `State/PfxRcd: 2` — session is **Established** (not "Idle" or "Active") and 2 EVPN routes have been received.
- `Up/Down` time = how long session has been up (or down). Stable session = increasing uptime.
- One neighbor entry per spine (both spines, since leaves peer with both).

**Bad output and what it means:**
- `State: Idle` — BGP can't establish. Usually `update-source loopback0` mismatch or BGP TCP port 179 blocked. Verify underlay loopback reachability.
- `State: Active` — BGP is trying to connect but failing. Same root causes as Idle.
- `State: OpenSent`/`OpenConfirm` stuck — handshake failing. AS number mismatch, password mismatch, or `nv overlay evpn` not configured.
- `State/PfxRcd: 0` — session up but no routes received. Often means `send-community extended` is missing somewhere, so route-targets aren't transmitted and routes aren't accepted.

### Per-route-type counters (bottom of summary output)
Neighbor   T   AS    Type-1  Type-2  Type-3  Type-4  Type-5  Type-6  Type-7  Type-8  Type-12
10.0.0.1   I   65000  0       1       1       0       0       0       0       0       0
10.0.0.2   I   65000  0       1       1       0       0       0       0       0       0

**What each type means in EVPN:**
- **Type-1** Ethernet Auto-Discovery — used in multi-homing scenarios (vPC, EVPN-MH). Empty in Phase 1.
- **Type-2** MAC / MAC+IP — the workhorse. Every host MAC and host MAC+IP advertised.
- **Type-3** Inclusive Multicast — "I'm a VTEP for VNI X, send BUM to me." Builds ingress-replication lists.
- **Type-4** Ethernet Segment — multi-homing.
- **Type-5** IP Prefix — for advertising whole subnets/external prefixes. Empty when you only have host /32s via Type-2.
- **Type-6** Selective Multicast — IGMP/PIM with EVPN.
- **Type-7/8** Multi-homing variants.
- **Type-12** ND — IPv6 ND in EVPN.

**In Phase 1 you should see only Type-2 and Type-3.** Anything else would be unusual.

### show bgp l2vpn evpn

The full EVPN BGP table. Long output; key fields:
Route Distinguisher: 10.0.0.11:32777    (L2VNI 10010)
*>l[2]:[0]:[0]:[48]:[aac1.ab90.48e2]:[32]:[10.10.10.10]/272
10.0.1.11    100   32768 i

**Decoding the route key `[2]:[0]:[0]:[48]:[aac1.ab90.48e2]:[32]:[10.10.10.10]/272`:**
- `[2]` = Route type 2 (MAC/MAC+IP)
- `[0]:[0]` = Ethernet Segment Identifier and Ethernet Tag (both 0 for single-homed simple deployments)
- `[48]` = MAC length in bits
- `[aac1.ab90.48e2]` = the MAC address
- `[32]` = IP length in bits (0 if MAC-only, 32 for IPv4, 128 for IPv6)
- `[10.10.10.10]` = the IP address
- `/272` = total NLRI length in bits

**Flag column on the left:**
- `*` = valid
- `>` = best path
- `l` = locally originated
- `i` = received from iBGP peer

**Two routes per host:**
1. `[2]:[0]:[0]:[48]:[MAC]:[0]:[0.0.0.0]/216` — MAC-only route, used for L2 forwarding within the L2VNI.
2. `[2]:[0]:[0]:[48]:[MAC]:[32]:[IP]/272` — MAC+IP route, used for L3 forwarding (symmetric IRB) and ARP suppression.

### show bgp l2vpn evpn route-type 2 (detailed)
BGP routing table entry for [2]:[0]:[0]:[48]:[aac1.ab90.48e2]:[32]:[10.10.10.10]/272
Path type: internal, path is valid, is best path
Imported to 3 destination(s)
Imported paths list: tenant1 L3-50001 L2-10010
10.0.1.11 (metric 81) from 10.0.0.1 (10.0.0.1)
Origin IGP, MED not set, localpref 100, weight 0
Received label 10010 50001
Extcommunity: RT:65000:10010 RT:65000:50001 ENCAP:8 Router MAC:0c71.fa00.1b08
Originator: 10.0.0.11 Cluster list: 10.0.0.1

**Critical fields:**

| Field | Meaning |
|---|---|
| `Received label 10010 50001` | Two labels = symmetric IRB. First is L2VNI, second is L3VNI. **Without both, symmetric IRB isn't working.** |
| `Extcommunity: RT:65000:10010` | MAC-VRF route target. Tells receiving leaf "import this into MAC-VRF for VNI 10010." |
| `Extcommunity: RT:65000:50001` | VRF route target. Tells receiving leaf "import this into VRF tenant1." |
| `ENCAP:8` | Encapsulation type 8 = VXLAN. |
| `Router MAC:0c71.fa00.1b08` | The source leaf's MAC. Used as the inner destination MAC when remote leaf routes traffic to this prefix. **Required for symmetric IRB.** |
| `Originator: 10.0.0.11` | The original sender. Always the leaf that owns the host. |
| `Cluster list: 10.0.0.1` | Which RR reflected the route. If you see 2 cluster list entries, both spines reflected — that's normal. |

**If `Received label` only has one value** = asymmetric IRB. Fix: re-check `member vni X associate-vrf` line on NVE.

**If `Router MAC` is missing** = symmetric IRB will fail. Usually means L3VNI isn't properly associated with the VRF.

### show bgp l2vpn evpn route-type 3
BGP routing table entry for [3]:[0]:[32]:[10.0.1.12]/88
10.0.1.12 (metric 81) from 10.0.0.1 (10.0.0.1)
Extcommunity: RT:65000:10010 ENCAP:8 PMSI: Flags:[0x0], Tunnel-Type:6, Label:10010

**What this tells you:** there's a remote VTEP at 10.0.1.12 that wants BUM for L2VNI 10010. Local leaf adds 10.0.1.12 to its ingress-replication list for VNI 10010.

Should see one Type-3 per remote VTEP per L2VNI.

---

## 8. Verification: VXLAN data plane

### show nve interface nve1 detail
Interface: nve1, State: Up, encapsulation: VXLAN
VPC Capability: VPC-VIP-Only [not-notified]
Local Router MAC: 0c71.fa00.1b08
Host Learning Mode: Control-Plane
Source-Interface: loopback1 (primary: 10.0.1.11, secondary: 0.0.0.0)
Source Interface State: Up
Fabric convergence time: 135 seconds
Fabric convergence time left: 0 seconds

**Critical fields:**
- `State: Up` — NVE interface is operational.
- `Host Learning Mode: Control-Plane` — EVPN, not flood-and-learn. **If this says "Data-Plane", `host-reachability protocol bgp` is missing.**
- `Source-Interface: loopback1 (primary: 10.0.1.11, secondary: 0.0.0.0)` — VTEP source IP. The `secondary` will hold the shared vPC-VTEP IP later in Phase 3.
- `Source Interface State: Up` — lo1 is reachable.
- `Fabric convergence time left: 0` — the leaf has finished initial fabric sync. New routes can be processed normally. If non-zero, the leaf is still in convergence mode (suppressing some routes during boot to avoid traffic loss).
- `Local Router MAC` — the MAC advertised in Type-2 routes as the Router MAC for L3-routed traffic.

### show nve peers
Interface  Peer-IP    State  LearnType  Uptime    Router-Mac
nve1       10.0.1.12  Up     CP         00:01:20  n/a

**Critical fields:**
- `State: Up` — VXLAN tunnel to this peer is operational.
- `LearnType: CP` — Control-Plane (BGP-EVPN). Confirms we're not flood-and-learn.
- One row per remote VTEP. Phase 1 has only one remote (the other leaf).

**Bad output:**
- Missing peer entirely = remote leaf hasn't advertised Type-3 yet, or Type-3 isn't being received. Check BGP-EVPN session and Type-3 routes.
- `State: Down` = underlay path to peer IP is broken. Check `show ip route <peer-IP>`.

### show nve vni
Interface  VNI    Multicast-group   State  Mode  Type             Flags
nve1       10010  UnicastBGP        Up     CP    L2 [10]
nve1       50001  n/a               Up     CP    L3 [tenant1]

**Critical fields:**
- `VNI 10010, Mode CP, Type L2 [10]` = L2VNI 10010 is bound to VLAN 10, control-plane learning.
- `VNI 50001, Mode CP, Type L3 [tenant1]` = L3VNI 50001 is bound to VRF tenant1.
- `Multicast-group: UnicastBGP` for L2VNI = ingress replication driven by BGP Type-3 routes.

**Bad output:**
- VNI in `Down` state = VLAN or VRF binding broken. Check `vn-segment` under VLAN and `vni` under VRF.
- L3VNI missing = `member vni X associate-vrf` not configured on NVE.

---

## 9. Verification: tenant L2 and L3

### show l2route mac all
Topology  Mac Address      Prod   Flags  Seq No  Next-Hops
10        aac1.ab15.ee88   BGP    Rcv    0       10.0.1.12 (Label: 10010)
10        aac1.ab90.48e2   Local  L,     0       Eth1/9

**Critical fields:**
- `Topology 10` = VLAN 10.
- `Prod: Local` = MAC learned locally on a switchport (host1 on leaf1).
- `Prod: BGP` = MAC learned via BGP-EVPN from a remote VTEP.
- `Next-Hops: 10.0.1.12 (Label: 10010)` = traffic to this MAC goes via VXLAN to VTEP 10.0.1.12 with VNI 10010.
- `Flags: L` = Local, `Rcv` = Received.

**This is the EVPN MAC table.** Every remote MAC should appear here with `Prod BGP`. If a remote MAC is missing, EVPN MAC learning failed somewhere.

### show mac address-table dynamic vlan 10
VLAN  MAC Address      Type     Ports
C 10  aac1.ab15.ee88   dynamic  nve1(10.0.1.12)

10  aac1.ab90.48e2   dynamic  Eth1/9


**Critical fields:**
- `nve1(10.0.1.12)` for the port = remote MAC, reached via NVE tunnel to that VTEP.
- `Eth1/9` for the port = locally-attached MAC on the host port.
- `C` flag (left column) = Control-Plane MAC (from EVPN).

This is the data-plane forwarding table that hardware/dataplane actually consults. EVPN populates it from the routes in `show l2route mac all`.

### show ip arp vrf tenant1
IP ARP Table for context tenant1
Total number of entries: 1
Address       Age       MAC Address     Interface
10.10.10.10   00:04:05  aac1.ab90.48e2  Vlan10

**Critical observation:** this is leaf1's ARP table. Only host1 (locally attached) is here. Host2 is NOT in this ARP table — it's in BGP-EVPN as a Type-2 route instead.

**This is how EVPN replaces fabric-wide ARP flooding:**
- Leaf1 ARPs for its local host → learns MAC → advertises Type-2 with MAC+IP → all leaves learn it via BGP.
- Leaf2 doesn't need to ARP for host1; it already has the MAC+IP from BGP.
- Result: ARP traffic stays local to each leaf, never floods the fabric.

### show ip route vrf tenant1
10.10.10.0/24, ubest/mbest: 1/0, attached
*via 10.10.10.1, Vlan10, [0/0], direct
10.10.10.1/32, ubest/mbest: 1/0, attached
*via 10.10.10.1, Vlan10, [0/0], local
10.10.10.10/32, ubest/mbest: 1/0, attached
*via 10.10.10.10, Vlan10, [190/0], hmm

**Critical fields:**
- `10.10.10.0/24, direct` = the connected subnet.
- `10.10.10.1/32, local` = the anycast gateway IP (same on all leaves).
- `10.10.10.10/32, hmm` = **Host Mobility Manager** route for a locally-attached host. NX-OS auto-installs /32 routes for locally-learned hosts. Helps with fast convergence when a host moves between leaves.

On leaf2 you'd see host2 as the `hmm` /32 and host1 as a BGP route:
10.10.10.10/32, ubest/mbest: 1/0
*via 10.0.1.11%default, [200/0], bgp-65000, internal, tag 65000, segid: 50001 tunnelid: 0xa00010b encap: VXLAN

**Decoding the BGP-VXLAN route:**
- `via 10.0.1.11%default` = next-hop is remote VTEP loopback in the default VRF (underlay).
- `[200/0]` = BGP administrative distance (200 for iBGP) / metric.
- `segid: 50001` = L3VNI used to encap the routed packet.
- `tunnelid: 0xa00010b` = internal tunnel identifier.
- `encap: VXLAN` = data-plane encapsulation type.

This is the data-plane realization of the BGP-EVPN Type-2 route for symmetric IRB.

### show ip interface brief vrf all

Quick sanity check. Should show:
VRF "default": loopbacks 0/1, all underlay interfaces (p2p IPs), mgmt0 in management VRF
VRF "tenant1": Vlan10 (10.10.10.1), Vlan100 (forward-enabled)

If anything's in the wrong VRF, that's a config bug.

### show vxlan — broken on NX-OSv
Traceback (most recent call last):
File "/isan/python3/scripts/vxlan_show.py", line 17, in <module>
ModuleNotFoundError: No module named 'tahoe'

Known NX-OSv bug. Has been broken across multiple versions. **Ignore it.** Use the other commands to get the same information:
- VXLAN status → `show nve interface nve1 detail`
- VNI table → `show nve vni`
- Tunnel peers → `show nve peers`

---

## 10. Verification: end-to-end traffic

### Ping host1 → host2
docker exec clab-phase1-baseline-host1 ping -c 4 10.10.10.20

Expected output:
PING 10.10.10.20 (10.10.10.20) 56(84) bytes of data.
64 bytes from 10.10.10.20: icmp_seq=1 ttl=64 time=99.2 ms
64 bytes from 10.10.10.20: icmp_seq=2 ttl=64 time=19.0 ms
64 bytes from 10.10.10.20: icmp_seq=3 ttl=64 time=17.9 ms
64 bytes from 10.10.10.20: icmp_seq=4 ttl=64 time=18.2 ms

**What's going on:**

| Ping | Time | What happened |
|---|---|---|
| icmp_seq=1 | 99.2ms | First ping. ARP must be resolved (or read from BGP-EVPN cache). MAC learning happens. |
| icmp_seq=2+ | 18-19ms | Steady state. Normal VXLAN-encapsulated path. |

**If ping fails:** start at the underlay (Section 6), then BGP (Section 7), then NVE (Section 8), then ARP/MAC tables (Section 9). The order matters — don't troubleshoot BGP if your loopback ping doesn't work.

---

## 11. Packet flow walkthrough

Host1 (10.10.10.10 on leaf1) pings host2 (10.10.10.20 on leaf2). Both in VLAN 10.

### Step 1 — Host1 builds the frame

Host1 sees destination 10.10.10.20 is in its own subnet (10.10.10.0/24). It needs host2's MAC address.

### Step 2 — ARP resolution

Host1 sends an ARP request: "Who has 10.10.10.20?" The frame is broadcast — destination MAC `ff:ff:ff:ff:ff:ff`.

The ARP request hits leaf1 on Eth1/9. Two things can happen:

**A. ARP suppression is on:** Leaf1 has host2's MAC+IP in its BGP-EVPN table (Type-2 route from leaf2). It locally responds with host2's MAC. ARP request never leaves leaf1.

**B. ARP suppression is off (our Phase 1):** Leaf1 floods the ARP broadcast into L2VNI 10010. Because we use ingress replication, leaf1 sends one VXLAN-encapsulated copy to each remote VTEP in the type-3 list (just leaf2 in our case). Leaf2 floods it into its VLAN 10. Host2 sees the ARP and replies.

Either way, host1 eventually has host2's MAC.

### Step 3 — Host1 sends ICMP

Host1 builds the Ethernet frame:
- src MAC: host1MAC
- dst MAC: host2MAC
- IP: src 10.10.10.10, dst 10.10.10.20
- ICMP echo request

Sent on eth1 → leaf1 Eth1/9.

### Step 4 — Leaf1 lookup

Leaf1 looks up host2MAC in its VLAN 10 MAC table. Finds:
host2MAC → nve1, VTEP 10.0.1.12, VNI 10010

### Step 5 — Leaf1 VXLAN encapsulation

Leaf1 wraps the original frame in:
- Outer Ethernet (leaf1's MAC → next-hop MAC toward 10.0.1.12)
- Outer IP: src 10.0.1.11, dst 10.0.1.12
- UDP: dst port 4789 (VXLAN)
- VXLAN header: VNI 10010
- Original frame (inner Ethernet + IP + ICMP)

### Step 6 — Underlay forwarding

Outer IP packet leaves leaf1, hits a spine. Spine looks up 10.0.1.12 in its IP routing table, finds OSPF route via Eth1/2 to leaf2. Forwards as plain IP.

Spine has **no idea** this is VXLAN. It's just routing IP packets.

### Step 7 — Leaf2 VXLAN decapsulation

Leaf2 receives the IP packet on its loopback1 (10.0.1.12). Sees UDP 4789 + VXLAN VNI 10010. Decaps.

Now it has the original frame: dst MAC host2MAC, in VLAN 10.

### Step 8 — Leaf2 forward to host2

Leaf2 looks up host2MAC in its VLAN 10 MAC table. Finds:
host2MAC → Eth1/9 (local)

Forwards on Eth1/9 to host2.

### Step 9 — Host2 ICMP reply

Host2 replies. The whole process runs in reverse: host2 → leaf2 → VXLAN encap → underlay → leaf1 → VXLAN decap → host1.

### Latency breakdown

- ARP resolution + MAC learning: ~80ms on first packet
- Subsequent packets: ~18ms (encap, underlay routing, decap, twice)

Steady-state 18ms is the actual latency of the fabric. 99ms first packet is one-time learning cost.

---

## 12. Common mistakes and their symptoms

### Mistake: `nv overlay evpn` missing on a node

**Symptom:** `address-family l2vpn evpn` rejected under `router bgp`. Or accepted but BGP session won't process EVPN routes.

**Fix:** add `nv overlay evpn` at global config on the affected node.

### Mistake: `send-community extended` missing

**Symptom:** BGP session established, but no routes get installed. Routes might appear in `show bgp l2vpn evpn` but flag column shows they're not best-path / not imported.

**Fix:** add `send-community extended` to the EVPN address family on the peer template. EVPN route-targets travel as extended communities — without this, RTs don't propagate and routes get dropped on import.

### Mistake: forgetting `feature interface-vlan` on leaf

**Symptom:** `interface Vlan10` command rejected.

**Fix:** `feature interface-vlan` at global config.

### Mistake: NVE without `host-reachability protocol bgp`

**Symptom:** Fabric falls back to data-plane learning (flood-and-learn). MAC tables fill up via broadcast, not BGP. `show nve interface nve1 detail` shows `Host Learning Mode: Data-Plane`.

**Fix:** add `host-reachability protocol bgp` under `interface nve1`.

### Mistake: TCAM reload trap on NX-OSv

**Symptom:** `hardware access-list tcam region arp-ether 256 double-wide` then `reload` leaves the leaf wedged. SSH dies, container appears "healthy" but unresponsive.

**Fix:** don't do this on NX-OSv. Either:
- Skip `suppress-arp` entirely (it's an optimization, not required).
- Bake the TCAM line into containerlab `startup-config` so it's applied at first boot (no reload needed).

### Mistake: anycast gateway IP/MAC differs across leaves

**Symptom:** Hosts get duplicate IP / duplicate MAC errors. VM mobility breaks.

**Fix:** verify `fabric forwarding anycast-gateway-mac 0000.2222.3333` and the SVI IP `10.10.10.1/24` are **identical** on every leaf.

### Mistake: L3VNI not associated with NVE

**Symptom:** Inter-subnet routing across the fabric fails. Within-subnet (L2) works fine.

**Fix:** add `member vni 50001 associate-vrf` under `interface nve1`.

### Mistake: feature `vn-segment-vlan-based` missing

**Symptom:** `vn-segment 10010` command rejected under VLAN.

**Fix:** `feature vn-segment-vlan-based` at global config.

---

## 13. Feature dependency cheat sheet

### Spine
feature ospf
feature bgp
nv overlay evpn

### Leaf
feature ospf
feature bgp
feature interface-vlan
feature vn-segment-vlan-based
feature nv overlay
nv overlay evpn

### Key reminders

- `nv overlay evpn` is a **separate global mode command**, not a `feature`. Easy to forget — required on both spines and leaves.
- `feature nv overlay` is only on leaves. Spines don't encap/decap VXLAN.
- `send-community extended` is non-optional in any EVPN address-family neighbor config. Forget it and route-targets don't propagate.

