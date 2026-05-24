# Phase 1: VXLAN-EVPN Fabric Reference Guide

A complete walkthrough of what we built in Phase 1, why each config piece is there, and how to read the show output. Come back here whenever you need to reground yourself.

---

## 1. The mental model

VXLAN-EVPN is two networks sharing the same physical hardware:

**Underlay** — the IP fabric. Just IP routing with OSPF. Job: get packets between leaf loopbacks.
- Carries: plain IP packets
- Knows nothing about tenants, MACs, VLANs, or VXLAN

**Overlay** — the tenant network. Lives inside the underlay as VXLAN tunnels.
- Carries: tenant Ethernet frames wrapped in VXLAN (UDP)
- Control plane: BGP with the L2VPN EVPN address family
- Knows about MAC addresses, IPs, VRFs, VLANs

### Why two device roles?

**Spines = transit + control plane.** They route IP packets in the underlay and reflect BGP-EVPN routes between leaves. They never originate tenant traffic, never have VRFs, never encap/decap VXLAN.

**Leaves = VTEPs (VXLAN Tunnel Endpoints).** Where hosts connect. Leaves encapsulate frames into VXLAN, decapsulate VXLAN into frames, have VRFs and SVIs for tenant L3, and run anycast gateway.

### Why spine-leaf (CLOS)?

Every leaf connects to every spine. Equal-cost multipath underlay, predictable latency, horizontal scalability. Adding more capacity = add more spines or more leaves; no architectural change.

---

## 2. Per-role config breakdown

### Spine config — what each line does
feature ospf            # underlay routing protocol
feature bgp             # control plane
nv overlay evpn         # enables L2VPN EVPN address family in BGP
interface loopback0
ip address 10.0.0.1/32                  # router-id for OSPF and BGP
ip router ospf 1 area 0.0.0.0           # advertise into OSPF
interface Ethernet1/1
no switchport                           # convert to L3 routed port
ip address 10.1.1.1/30                  # p2p underlay link
ip router ospf 1 area 0.0.0.0
no shutdown
router bgp 65000
router-id 10.0.0.1
address-family l2vpn evpn               # global activation
template peer LEAF-RR-CLIENT
remote-as 65000                       # same AS = iBGP
update-source loopback0               # source BGP from stable loopback
address-family l2vpn evpn
send-community                      # standard communities
send-community extended             # extended (route targets live here)
route-reflector-client              # spine reflects between leaves
neighbor 10.0.0.11 inherit peer LEAF-RR-CLIENT
neighbor 10.0.0.12 inherit peer LEAF-RR-CLIENT

### Why route-reflector?

iBGP rule: a router does NOT re-advertise routes learned from one iBGP peer to another iBGP peer. Without a route reflector, leaf1 and leaf2 would never learn each other's routes through the spine — the spine would receive them but not forward. Route reflector breaks that rule for its clients (the leaves).

### What spines DON'T have

- No NVE interface (no VXLAN encap/decap)
- No VRFs (no tenant L3)
- No SVIs (no host-facing L3)
- No VLAN-to-VNI mappings
- No `feature nv overlay`, no `feature vn-segment-vlan-based`

The spine is essentially a high-speed IP router with one extra job: reflect EVPN routes.

### Leaf config — what each line does
feature ospf                            # underlay
feature bgp                             # control plane
feature interface-vlan                  # needed for SVIs
feature vn-segment-vlan-based           # allows "vn-segment XXXX" under a VLAN
feature nv overlay                      # VXLAN encap/decap engine
nv overlay evpn                         # L2VPN EVPN address family
interface loopback0
ip address 10.0.0.11/32               # router-id
ip router ospf 1 area 0.0.0.0
interface loopback1
description VTEP-source               # THIS is the VTEP IP
ip address 10.0.1.11/32               # remote leaves use this as tunnel destination
ip router ospf 1 area 0.0.0.0         # advertise so it's reachable
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
vlan 100
name L3VNI-tenant1
vn-segment 50001                      # this VLAN is the L3VNI transit VLAN
vrf context tenant1
vni 50001                             # the L3VNI for this tenant VRF
rd auto
address-family ipv4 unicast
route-target both auto
route-target both auto evpn         # auto RT for EVPN
interface Vlan100
no shutdown
mtu 9216
vrf member tenant1
ip forward                            # L3VNI SVI just forwards; no IP needed
vlan 10
name HOST_VLAN
vn-segment 10010                      # this VLAN extends as L2VNI 10010
evpn
vni 10010 l2
rd auto                             # MAC-VRF RD
route-target import auto
route-target export auto
fabric forwarding anycast-gateway-mac 0000.2222.3333    # same MAC on all leaves
interface Vlan10
no shutdown
mtu 9216
vrf member tenant1
no ip redirects
ip address 10.10.10.1/24              # anycast gateway IP
fabric forwarding mode anycast-gateway
interface nve1                          # VXLAN tunnel interface
no shutdown
host-reachability protocol bgp        # use BGP-EVPN, not flood-and-learn
source-interface loopback1            # tunnel source
member vni 10010
ingress-replication protocol bgp    # BUM via unicast copies (BGP type-3 builds list)
member vni 50001 associate-vrf        # L3VNI for routed tenant traffic
interface Ethernet1/9
switchport                            # L2 access port
switchport mode access
switchport access vlan 10
no shutdown

---

## 3. The concepts you just touched

### L2VNI vs L3VNI

**L2VNI (10010)** bridges a VLAN across leaves. Two hosts in the same VLAN on different leaves act like they're on the same Ethernet segment. Traffic between them is bridged.

**L3VNI (50001)** routes between subnets in the same VRF. When host1 sends to a host in a different subnet on leaf2:
1. Packet is routed at leaf1 (ingress routing) using the anycast gateway
2. Encapsulated into VXLAN with L3VNI in the header
3. Decapsulated at leaf2
4. Routed at leaf2 (egress routing) to the destination subnet
5. Bridged to the destination host

This is **symmetric IRB** — both ingress and egress leaves do L3. Asymmetric IRB only routes at one side and breaks at scale.

### Anycast gateway

Both leaves have the same SVI IP `10.10.10.1` with the same MAC `0000.2222.3333`. Host1 and Host2 both think their gateway is `10.10.10.1` regardless of which leaf they're connected to. When a host moves between leaves (VM migration), nothing changes from the host's view.

### Ingress replication vs multicast underlay

Two ways to handle BUM (Broadcast, Unknown unicast, Multicast):

- **Multicast underlay** — every leaf joins a multicast group; BUM sent once. Requires PIM.
- **Ingress replication (what we use)** — sending leaf makes one unicast copy per remote VTEP. Simpler — no multicast in underlay. Scales fine up to hundreds of VTEPs.

BGP-EVPN Type-3 routes advertise "I'm a VTEP for VNI X, send BUM to me here." That's how ingress-replication lists are built dynamically.

---

## 4. Reading the show output

### show bgp l2vpn evpn summary

BGP peering state.
Neighbor   V   AS   MsgRcvd  MsgSent  ...  Up/Down  State/PfxRcd
10.0.0.1   4   65000 8       11            00:00:04 2

`State/PfxRcd = 2` means session is Established and 2 EVPN routes received. If you see `Idle` or `Active`, session isn't up. The per-type counter table below shows how many of each EVPN route type was received.

### show nve peers

The data-plane peer list — other VTEPs we can VXLAN-tunnel to.
Interface  Peer-IP    State  LearnType  Uptime
nve1       10.0.1.12  Up     CP         00:01:20

`LearnType: CP` = control-plane (BGP-EVPN). `DP` would be legacy flood-and-learn.

### show nve vni

Per-VNI state.
Interface  VNI    Multicast-group  State  Mode  Type
nve1       10010  UnicastBGP       Up     CP    L2 [10]
nve1       50001  n/a              Up     CP    L3 [tenant1]

`Multicast-group: UnicastBGP` = ingress replication driven by BGP type-3 routes.

### show bgp l2vpn evpn route-type 2

The heart of EVPN. Type-2 carries MAC and MAC+IP routes.

Route key format: `[2]:[0]:[0]:[48]:[MAC]:[IP-length]:[IP]/length`
- `[2]` = route type 2
- `[48]` = MAC length (48 bits)
- `[32]` = IPv4 length (or `[0]` for MAC-only)

Labels:
- `Received label 10010` = L2VNI only (MAC-only route)
- `Received label 10010 50001` = L2VNI + L3VNI (MAC+IP route, symmetric IRB)

Extcommunities:
- `RT:65000:10010` = MAC-VRF route target
- `RT:65000:50001` = VRF route target
- `ENCAP:8` = VXLAN encapsulation
- `Router MAC:xxxx.xxxx.xxxx` = source leaf's MAC, used as inner dst MAC for L3 traffic

Two labels + Router MAC = symmetric IRB is working.

### show bgp l2vpn evpn route-type 3

Inclusive Multicast Ethernet Tag — "I'm a VTEP for VNI X, send BUM here." Builds ingress-replication lists.

### show bgp l2vpn evpn route-type 5

IP prefix routes. Empty when you only have host /32 routes from Type-2. Shows up with external prefix advertisement or inter-VRF leaking.

### show l2route mac all

MAC table after EVPN merge.

- `Prod: Local, Flags: L` = locally-learned on a switchport
- `Prod: BGP, Flags: Rcv` = received via BGP-EVPN from remote VTEP
- `Label: 10010` = the L2VNI for VXLAN encap

### show ip arp vrf tenant1

Hosts directly attached to THIS leaf's anycast gateway. **Remote hosts are NOT here** — they're in the BGP-EVPN table. Each leaf ARPs only for its local hosts. This is what "ARP suppression by EVPN" means.

### show ip route vrf tenant1

Per-tenant routing table.

Local host:
10.10.10.10/32, attached
*via 10.10.10.10, Vlan10, [190/0], hmm
`hmm` = Host Mobility Manager — locally-learned hosts get a /32 for fast convergence.

Remote host:
10.10.10.10/32
*via 10.0.1.11%default, [200/0], bgp-65000, segid: 50001 tunnelid: 0xa00010b encap: VXLAN
Next-hop is remote VTEP loopback1. segid = L3VNI. Data-plane realization of the Type-2 EVPN route.

### show vxlan (broken on NX-OSv)

Throws a python ModuleNotFoundError. Known NX-OSv bug. Ignore — use the other commands.

### show ip interface brief vrf all

Sanity check that interfaces are in the right VRF: loopbacks in default, mgmt0 in management, SVIs in tenant1.

---

## 5. What happens when host1 pings host2

In sequence:

1. Host1 sends ICMP to 10.10.10.20. Same subnet → needs Host2's MAC.
2. Host1 ARPs for 10.10.10.20. Leaf1 has Host2's MAC pre-installed from BGP-EVPN Type-2 routes — responds locally (or floods only on first miss).
3. Host1 sends Ethernet frame: src=Host1MAC, dst=Host2MAC.
4. Leaf1 MAC table says Host2MAC → nve1 with L2VNI 10010 to VTEP 10.0.1.12.
5. Leaf1 wraps frame in VXLAN: VNI 10010, src IP 10.0.1.11, dst IP 10.0.1.12.
6. Underlay (OSPF) routes the IP packet from 10.0.1.11 to 10.0.1.12 via a spine.
7. Leaf2 receives, decapsulates, sees VNI 10010 = VLAN 10. Forwards frame to Host2 on Eth1/9.
8. ICMP reply follows the reverse path.

18ms steady-state = this whole loop, twice (ping + reply).
99ms first ping = ARP resolution + EVPN learn on first miss.

---

## 6. Mistakes we made (and lessons)

### 1. `reload` after TCAM change broke the leaf

Don't do this on NX-OSv under nested KVM. Reload can leave the container wedged. Either bake config into containerlab startup-config, or skip features that require reload.

### 2. Forgot `nv overlay evpn` on spines

Without it, the `address-family l2vpn evpn` knob doesn't appear in `router bgp`. Required on BOTH spines and leaves. Easy to miss because most VXLAN config lives on leaves only.

### 3. `suppress-arp` requires TCAM region

`suppress-arp` needs `hardware access-list tcam region arp-ether 256 double-wide`, which requires reload. We skipped suppress-arp entirely — it's an optimization, not a requirement. Phase 1 works fine without it. Add later if needed.

---

## 7. Feature dependency cheat sheet

**Spine:**
- feature ospf
- feature bgp
- nv overlay evpn

**Leaf (full VTEP):**
- feature ospf
- feature bgp
- feature interface-vlan
- feature vn-segment-vlan-based
- feature nv overlay
- nv overlay evpn

Note: `nv overlay evpn` is NOT a `feature` command — it's a separate global mode switch. Easy to forget.

---

## 8. Topology recap
              spine1                spine2
          10.0.0.1               10.0.0.2
          AS 65000 (RR)          AS 65000 (RR)
            /    \                 /    \
    10.1.1  /     \ 10.1.2  10.1.3 /     \ 10.1.4
          /        \             /        \
       leaf1                              leaf2
     10.0.0.11                          10.0.0.12
     lo1: 10.0.1.11                     lo1: 10.0.1.12
        |                                  |
     Eth1/9                              Eth1/9
        |                                  |
     host1                               host2
   10.10.10.10                         10.10.10.20
VLAN 10  -> L2VNI 10010
VRF tenant1 -> L3VNI 50001
Anycast GW: 10.10.10.1 / 0000.2222.3333

