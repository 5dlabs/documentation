# Blockchain Node Specifications

> Managed via [5dlabs/kotal](https://github.com/5dlabs/kotal) fork — deployed as Kubernetes CRDs on the CTO cluster.
> Kotal operator is running in the `kotal` namespace on OVH BHS5.

---

## Overview

```mermaid
flowchart TB
    subgraph KOTAL["Kotal Operator (kotal namespace — OVH BHS5)"]
        OP["kotal-operator\nghcr.io/5dlabs/kotal:latest\n(nearcore 2.10.6 + reth fork)"]
    end

    subgraph NEAR_STACK["NEAR Mainnet Node"]
        NEAR_CRD["near.kotal.io/v1alpha1\nNode CR"]
        NEAR_SVC["NEAR RPC\nnearcore 2.10.6\nJSON-RPC :3030\nNetwork: mainnet"]
        NEAR_VOL["PVC\n800 GB NVMe\n(t1-90 local disk)"]
        NEAR_CRD --> NEAR_SVC
        NEAR_SVC --> NEAR_VOL
    end

    subgraph BASE_STACK["BASE Mainnet Node (Reth)"]
        BASE_CRD["ethereum.kotal.io/v1alpha1\nNode CR"]
        BASE_SVC["BASE RPC\nclient: reth\nnetwork: base\nJSON-RPC :8545\nWebSocket :8546"]
        BASE_VOL["PVC\n1.8 TB total\n800 GB NVMe + 1 TB block vol"]
        BASE_CRD --> BASE_SVC
        BASE_SVC --> BASE_VOL
    end

    OP -->|"manages"| NEAR_CRD
    OP -->|"manages"| BASE_CRD
```

---

## NEAR Mainnet

### Hardware Requirements

| Resource | Minimum | Recommended |
|---|---|---|
| vCPU | 8 | 16 |
| RAM | 24 GB | 64 GB+ |
| Storage | 500 GB NVMe | 1 TB NVMe |
| Network | 1 Gbps | 1 Gbps+ |

### OVH Instance: `t1-90`

| Property | Value |
|---|---|
| vCPU | 16 |
| RAM | 90 GB |
| Disk | 800 GB NVMe |
| Network | Up to 1 Gbps |
| Region | BHS5 |

### Kotal CRD (example)

```yaml
apiVersion: near.kotal.io/v1alpha1
kind: Node
metadata:
  name: near-mainnet
  namespace: blockchain
spec:
  network: mainnet
  nodePrivateKeySecretName: near-node-key
  rpc: true
  resources:
    cpu: "8"
    memory: "32Gi"
    storage: "750Gi"
    storageClass: "local-nvme"
```

---

## BASE Mainnet (via Reth)

### Hardware Requirements

| Resource | Minimum | Recommended |
|---|---|---|
| vCPU | 8 | 16+ |
| RAM | 32 GB | 64 GB+ |
| Storage | 1.5 TB NVMe | 2 TB+ NVMe |
| Network | 1 Gbps | 1 Gbps+ |

> BASE is an OP Stack L2. Reth syncs both the L2 chain and reads L1 (Ethereum mainnet) state.
> Storage grows ~500 GB/year for archive, ~250 GB/year for pruned.

### OVH Instance Options

| Instance | vCPU | RAM | Disk | Notes |
|---|---|---|---|---|
| `t1-90` | 16 | 90 GB | 800 GB NVMe | Add 1 TB OVH block vol for archive |
| `t2-90` | 30 | 90 GB | 800 GB NVMe | More CPU for faster sync |
| `t1-180` | 32 | 180 GB | 400 GB NVMe | ⚠️ Less disk — needs 2×1 TB block vols |
| `t1-le-90` | 16 | 90 GB | 400 GB NVMe | ❌ Too small |

**Recommended: `t1-90` + 1 TB OVH block volume**

### Kotal CRD (example)

```yaml
apiVersion: ethereum.kotal.io/v1alpha1
kind: Node
metadata:
  name: base-mainnet
  namespace: blockchain
spec:
  network: base
  client: reth
  rpc: true
  ws: true
  resources:
    cpu: "8"
    memory: "32Gi"
    storage: "1500Gi"
    storageClass: "local-nvme"
```

---

## Solana RPC Node (External — PhoenixNAP Singapore)

> Not managed by Kotal. Standalone bare-metal deployment.

```mermaid
flowchart LR
    subgraph SOL_PHXNAP["PhoenixNAP Singapore — 125.253.92.141"]
        direction TB
        BIN["Jito v3.1.8-jito\n(client: Bam)\n/home/ubuntu/jito-solana/"]
        GRPC["yellowstone-grpc v12\ngRPC :10000\nGeyser plugin"]
        RPC["JSON-RPC :8899"]
        subgraph DISKS["Disk Layout (post-migration Feb-24)"]
            NVME0["nvme0n1 (root)\nOS + /ledger (637 GB)\n+ swap (200 GB)"]
            NVME1["nvme1n1 (/solana)\naccounts (513 GB)\nsnapshots (484 GB)\nswap 72 GB"]
        end
        BIN --> RPC
        BIN --> GRPC
        BIN --> DISKS
    end

    WATCHDOG["solana-watchdog.sh\nChecks RPC health every 60s\nRestarts after 5 fails"]
    WATCHDOG -->|"monitors"| RPC
```

### Key Configuration

| Setting | Value | Reason |
|---|---|---|
| `vm.swappiness` | 1 | Prevents swap thrashing |
| `--ledger` | `/ledger` (root disk) | Isolated I/O from accounts |
| `--accounts-dir` | `/solana/accounts` | Separate NVMe |
| `--snapshot-interval-slots` | 5000 | Reduce snapshot I/O |
| `--no-rocksdb-compaction` | removed in v3.1.8 | Not supported |
| Secondary indexes | disabled | Saves 60–80 GB RAM |
| yellowstone-grpc | v12.1.0-rc5 | gRPC streaming |

---

## Cluster Capacity: OVH BHS5 Quota

> **Current state:** 32 / 34 cores used. **Quota increase required before adding new VMs.**

```mermaid
pie title BHS5 vCPU Usage (34 core quota)
    "cto-node-1 (control-plane)" : 8
    "cto-node-2 (worker)" : 8
    "cto-node-3 (worker)" : 8
    "cto-node-4 (worker)" : 8
    "Available headroom" : 2
```

### To provision NEAR + BASE nodes:
- **Need ~32 additional cores** (2× t1-90 = 32 vCPU)
- **Action required:** Request OVH BHS5 quota increase to ≥80 cores
  - OVH Console → Project `6093a51de65b458e8b20a7c570a4f2c1` → Quota → Request increase
  - Justify: blockchain RPC nodes for production Web3 infrastructure
