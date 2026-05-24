# Phase 1: VXLAN-EVPN Fabric Baseline

## Goals
- Build a minimal but realistic 2-spine, 2-leaf VXLAN-EVPN fabric
- Underlay: OSPF area 0 with /30 p2p links
- Overlay: iBGP-EVPN in AS 65000, spines as route reflectors
- Tenant: one VRF (tenant1), one VLAN (10), symmetric IRB
- Anycast gateway on both leaves
- Two Linux hosts in VLAN 10 — one per leaf
- **Success: Host1 pings Host2 across the fabric via VXLAN encapsulation**

## Why this matters
This is the foundation every later phase builds on. vPC (Phase 3), multi-pod
(Phase 4), and multi-site (Phase 5) all assume a working single-fabric design
like this. Get this rock-solid before moving on.

## Topology
See `topology/phase1.clab.yml`.

## Build sequence
1. Deploy fabric → wait for all nodes healthy
2. Configure underlay (loopbacks + p2p IPs + OSPF) → verify OSPF neighborships
3. Configure overlay (iBGP-EVPN with spines as RRs) → verify BGP neighborships
4. Configure VRF + L3VNI
5. Configure VLAN 10 + L2VNI + NVE interface
6. Configure anycast gateway SVI
7. Configure hosts → ping test

## Success criteria
- `show ip ospf neighbors` shows 2 FULL adjacencies on each leaf
- `show bgp l2vpn evpn summary` shows 2 BGP neighbors UP on each leaf
- `show nve peers` shows the other leaf as a peer
- `show l2route mac all` shows remote MACs learned via EVPN
- Host1 (10.10.10.10) pings Host2 (10.10.10.20)
