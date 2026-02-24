# 5DLabs Infrastructure Documentation

Architecture and capacity planning for the CTO platform on OVH Public Cloud.

> **Scope:** OVH infrastructure only — CTO cluster, GPU compute, and blockchain nodes.

## Contents

- [CTO Platform Architecture](./cto-platform-architecture.md) — OVH BHS5 cluster overview
- [GPU Capacity Planning](./gpu-capacity-planning.md) — MiniMax-Text-01 requirements + OVH GPU inventory
- [Blockchain Node Specs](./blockchain-nodes.md) — NEAR mainnet + BASE mainnet via Kotal

## OVH Region Summary

| Region | Location | What's Here |
|---|---|---|
| **BHS5** | Beauharnois, Canada 🇨🇦 | CTO cluster (4× b2-30), blockchain NVMe VMs (planned) |
| **GRA9/GRA11** | Gravelines, France 🇫🇷 | All GPU instances (A100, H100, L4, L40S) |

> ⚠️ **GPU availability gap:** OVH does not offer GPU instances in any Canadian region.
> GPU compute (MiniMax-Text-01) must run in GRA with cross-region connectivity to BHS5,
> or be deferred until OVH expands GPU capacity to BHS5.
