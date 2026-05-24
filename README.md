# VXLAN-EVPN Lab Journey

Hands-on containerlab labs for Cisco NX-OSv 10.5.5, progressing from single-fabric
VXLAN-EVPN through vPC, multi-pod, and multi-site designs.

## Environment
- ESXi 7.0 host: Dell Precision T5600, 2× Xeon E5-2660 (16C/32T), 128 GB RAM
- Lab VM: Ubuntu 24.04 LTS, 24 vCPU, 96 GB RAM, 250 GB disk
- Containerlab + Docker + nested KVM
- NX-OSv 9300v 10.5.5 (cisco_n9kv container)

## Phases
- [x] Phase 1: Single fabric, anycast gateway, symmetric IRB
- [x] Phase 2 Pattern 1: Trunk/hypervisor port
- [ ] **Phase 3: vPC (current)**
- [ ] Phase 4: Multi-pod
- [ ] Phase 5: Multi-site (BGW, anycast BGW)
- [ ] Phase 6: NDFC overlay (DevNet sandbox)

## Repository structure
Each phase is self-contained with its own topology, configs, verification, and notes.
