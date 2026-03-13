# KBERDL CPU/RAM Capacity Analysis

---

## 1. Current State

### 1.1 Cluster Node Inventory

| Node | Cores | RAM | Disk | Current Assignment |
|------|------:|----:|-----:|-------------------|
| kworker-03 | 168 | 1024 GB | 100 TB | **All user notebook pods** |
| kworker-05 – 10 | 168 each | 1024 GB each | 100 TB each | Per-user Spark clusters (master + workers) |
| kworker-11 | 168 | 1024 GB | 100 TB | Shared Spark cluster (master + workers) |
| kworker-12 – 13 | 168 each | 1024 GB each | 100 TB each | **Other workloads** |
| kworker-14 – 17 | 168 each | 1024 GB each | 100 TB each | Per-user Spark clusters (master + workers) |
| kworker-18 | 168 | 1024 GB | 100 TB | Shared Spark cluster (workers) |
| kworker-19 – 20 | 168 each | 1024 GB each | 100 TB each | **Other workloads** |

**Spark cluster pool: 10 nodes** (kworker-05–10, kworker-14–17) — **1,680 cores, 10,240 GB RAM total**

### 1.2 User Workload Components

Each user session spawns **three components** across two node groups (managed by [BERDL_JupyterHub](https://github.com/BERDataLakehouse/BERDL_JupyterHub)):

```
┌──────────────────────────────────────────────────────────────┐
│  kworker-03 (168 cores, 1024 GB RAM, 100 TB disk)            │
│                                                              │
│  ┌─────────────────┐                                         │
│  │  Notebook Pod   │  (All users scheduled here)             │
│  │  (JupyterLab +  │                                         │
│  │  Spark Connect  │                                         │
│  │  server inside) │                                         │
│  └─────────────────┘                                         │
│                                                              │
│  User home dirs stored on local disk via hostPath            │
│  Path: /mnt/state/prod/hub_homes/{username}                  │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  kworker-05–10, 14–17 (per-user Spark clusters)              │
│                                                              │
│  ┌────────────────┐  ┌──────────────┐                        │
│  │  Spark Master  │  │ Spark Workers│  (per-user,            │
│  │  (per-user)    │  │  5 or 10     │   dynamically created) │
│  └────────────────┘  │  replicas)   │                        │
│                      └──────────────┘                        │
└──────────────────────────────────────────────────────────────┘
```

### 1.3 Profile Definitions

Two profiles are available to users. Both profiles use **identical notebook pods** — the differentiation is entirely in the Spark cluster size.

#### Notebook Pod (both profiles) — [spark_notebook](https://github.com/BERDataLakehouse/spark_notebook)

| Component | CPU (guarantee) | Memory (limit) | Memory (guarantee) |
|-----------|----------------:|---------------:|-------------------:|
| Notebook pod | 1 core | 24 GB | 6 GB |

> No `cpu_limit` is set — pods can burst to use available CPU on the node.

#### Medium Spark Cluster

| Component | Cores | Memory |
|-----------|------:|-------:|
| Spark Master (×1) | 1 | 2 GiB |
| Spark Workers (×5) | 3 each (15 total) | 20 GiB each (100 GiB total) |
| **Per-user Spark total** | **16 cores** | **102 GiB** |

> 3 cores/worker is the sweet spot for Spark executors (balances parallelism vs GC pressure). 20 GiB/worker provides 6.7 GB/core — generous for data processing.

#### Large Spark Cluster

| Component | Cores | Memory |
|-----------|------:|-------:|
| Spark Master (×1) | 2 | 8 GiB |
| Spark Workers (×10) | 6 each (60 total) | 40 GiB each (400 GiB total) |
| **Per-user Spark total** | **62 cores** | **408 GiB** |

> Large is 4× the memory and ~4× the cores of Medium, designed for power-user workloads.

---

## 2. Notebook Pod Capacity (kworker-03)

### 2.1 Available Resources

| Resource | Total | System/OS Overhead (~5%) | Available for Workloads |
|----------|------:|-------------------------:|------------------------:|
| CPU | 168 cores | ~8 cores | **~160 cores** |
| RAM | 1024 GB | ~50 GB | **~974 GB** |

### 2.2 Maximum Concurrent Users

**Per-user notebook pod:** 1 core (guarantee), 24 GB RAM (limit), 6 GB RAM (guarantee), no CPU limit

> [!NOTE]
> Spark Master and Workers are scheduled on kworker-05–10 and kworker-14–17, **not** kworker-03. The capacity analysis below only considers notebook pod resources on kworker-03.

| Bottleneck | Available | Per User | Max Users |
|-----------|----------:|---------:|----------:|
| CPU (by guarantee) | 160 cores | 1 core | **160 users** |
| RAM (by guarantee) | 974 GB | 6 GB | **162 users** |
| RAM (by limit) | 974 GB | 24 GB | 40 users |

> **Result: kworker-03 can schedule up to ~160 concurrent notebook pods** (CPU guarantee is the binding constraint).
>
> RAM overcommit ratio is 4× (24 GB limit / 6 GB guarantee). At 100 users, total guaranteed RAM = 600 GB (62% of 974 GB). Total limit = 2,400 GB — typical JupyterLab + Spark Connect client pods rarely approach their memory limit, so this overcommit is acceptable.

---

## 3. Per-User Spark Cluster Capacity (kworker-05–10, 14–17)

Per-user Spark clusters (managed by [spark_cluster_manager](https://github.com/BERDataLakehouse/spark_cluster_manager)) run on **10 dedicated nodes** (kworker-05 through kworker-10, kworker-14 through kworker-17). Each node has 168 cores and 1024 GB RAM.

**Total Spark pool: 1,680 cores, 10,240 GB RAM**

### 3.1 Medium Spark Cluster

| Component | Cores | Memory |
|-----------|------:|-------:|
| Spark Master (×1) | 1 | 2 GiB |
| Spark Workers (×5) | 3 each (15 total) | 20 GiB each (100 GiB total) |
| **Per-user total** | **16 cores** | **102 GiB** |

| Bottleneck | Total Pool | Per User | Max Users |
|-----------|----------:|---------:|----------:|
| CPU | 1,680 cores | 16 cores | **105 users** |
| RAM | 10,240 GB | 102 GB | **100 users** |

> **Result: 10 Spark nodes can serve up to ~100 concurrent Medium Spark clusters** (RAM is the bottleneck).
>
> At 100 users: 10,200 GiB (99.6% RAM), 1,600 cores (95.2% CPU) — both resources near full utilization.

### 3.2 Large Spark Cluster

| Component | Cores | Memory |
|-----------|------:|-------:|
| Spark Master (×1) | 2 | 8 GiB |
| Spark Workers (×10) | 6 each (60 total) | 40 GiB each (400 GiB total) |
| **Per-user total** | **62 cores** | **408 GiB** |

| Bottleneck | Total Pool | Per User | Max Users |
|-----------|----------:|---------:|----------:|
| CPU | 1,680 cores | 62 cores | **27 users** |
| RAM | 10,240 GB | 408 GB | **25 users** |

> **Result: 10 Spark nodes can serve up to ~25 concurrent Large Spark clusters** (RAM is the bottleneck).

---

## 4. Combined Capacity Summary

| Profile | Notebook Capacity (kworker-03) | Spark Cluster Capacity (10 nodes) | Effective Limit |
|---------|-------------------------------:|----------------------------------:|----------------:|
| Medium | 160 users | 100 users | **100 users** (Spark-limited) |
| Large | 160 users | 25 users | **25 users** (Spark-limited) |

---

## 5. Shared Spark Cluster (kworker-11 + kworker-18)

A shared Spark cluster spans **2 nodes** for multi-tenant workloads (e.g., data API, event processor, ad-hoc queries).

### 5.1 Cluster Configuration

| Component | Cores | Memory | Replicas | Node |
|-----------|------:|-------:|---------:|------|
| Spark Master | 1 | 2 GiB | 1 | kworker-11 |
| Spark Workers | 8 each | 50 GiB each | 40 | kworker-11 + kworker-18 |
| **Total** | **321** | **2,002 GiB** | | |

> Workers use `nodeAffinity` to schedule across both kworker-11 and kworker-18. Each node fits ~20 workers (20 × 8c = 160c, 20 × 50G = 1,000G per node).
>
> 8 cores/worker allows Spark to run 2 executors of 4 cores each per worker — staying within the recommended 2–5 cores/executor sweet spot.

### 5.2 Resource Utilization

| Resource | Total (2 nodes) | Used | Utilization |
|----------|----------------:|-----:|------------:|
| CPU | 336 cores | 321 cores | **95.5%** |
| RAM | 2,048 GB | 2,002 GiB | **97.8%** |

### 5.3 Data API Executor Configuration

The [datalake-mcp-server](https://github.com/BERDataLakehouse/datalake-mcp-server) (data API) connects to the shared cluster. Each data API Spark session requests:

| Setting | Value | Description |
|---------|-------|-------------|
| `SPARK_WORKER_CORES` | 2 | Cores per executor |
| `SPARK_WORKER_COUNT` | 2 | Executors per job |
| `SPARK_WORKER_MEMORY` | 16 GiB | Memory per executor |
| `STANDALONE_SPARK_POOL_SIZE` | 8 | Concurrent sessions per replica |
| **Per-job total** | **4 cores, 32 GiB** | 2 executors × (2c, 16G) |

**Executor packing per worker (8c, 50G):**
- CPU: 8 / 2 = 4 executors
- RAM: 50 / 16 = 3 executors
- **3 executors per worker** (memory-bound) → 40 workers × 3 = **120 executor slots**
- 120 slots / 2 executors per job = **60 max concurrent jobs**

**Data API concurrency:** 8 replicas × pool size 8 = 64 max sessions (≥ 60)

| At 60 concurrent jobs | Used | Available | Utilization |
|-----------------------|-----:|----------:|------------:|
| CPU | 240 cores | 320 cores | **75%** |
| RAM | 1,920 GiB | 2,000 GiB | **96%** |

---
