# QAP Deployment Guide

**Author:** CHALLAYOGESWAR (challayogeswar8790@gmail.com)
**Version:** 1.0 | **Date:** 21 May 2026

---

## Prerequisites

| Requirement | Minimum | Recommended |
|---|---|---|
| Node hardware | TPM 2.0 chip | TPM 2.0 + HSM |
| OS | Linux (kernel ≥ 5.8 for eBPF) | Ubuntu 22.04 LTS / RHEL 9 |
| Orchestrator instances | 3 | 5 |
| Network controller | OpenDaylight 0.18+ | ONOS 2.7+ |
| TLS version | 1.2 | 1.3 |

---

## Component Stack

```
┌─────────────────────────────────────────────────────┐
│              QAP Orchestrator Cluster               │
│         (Kubernetes + Raft, min 3 nodes)            │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │  Vault       │  │  Kafka       │  │  RL Engine│ │
│  │  (key mgmt)  │  │  (ledger)    │  │  (RLlib)  │ │
│  └──────────────┘  └──────────────┘  └───────────┘ │
└────────────────────────┬────────────────────────────┘
                         │ mTLS (TLS 1.3)
         ┌───────────────┼───────────────┐
         │               │               │
    ┌────▼───┐      ┌────▼───┐      ┌────▼───┐
    │ Node A │      │ Node B │      │ Decoy  │
    │ ACTIVE │      │QUIESCE │      │Honeypot│
    │ eBPF ✓ │      │ eBPF ✓ │      │        │
    └────────┘      └────────┘      └────────┘
         │
    ┌────▼────────┐
    │ SDN         │
    │ Controller  │  ← OpenDaylight / ONOS
    └─────────────┘
```

---

## Step-by-Step Deployment

### Step 1 — Enroll nodes
- Provision TPM 2.0 endorsement keys for each node
- Register node identity (public key fingerprint) with the Orchestrator
- Install Orchestrator agent and eBPF programs on each node

### Step 2 — Deploy Orchestrator cluster
- Deploy 3+ Orchestrator instances via Kubernetes
- Configure Raft consensus between instances
- Initialize HashiCorp Vault for signing key management
- Deploy Apache Kafka cluster for the trust ledger

### Step 3 — Configure SDN controller
- Connect OpenDaylight/ONOS to the Orchestrator MTD engine
- Define IP pool and port range for rotation
- Test rotation with a single node before full deployment

### Step 4 — Deploy decoy honeypots
- Deploy minimum 2 decoy nodes per 10 active nodes
- Connect honeypot interaction logs to the Orchestrator threat intelligence feed

### Step 5 — Validate
- Run chaos engineering simulation: induce QUIESCE claims, test quorum rejection,
  simulate compromise and verify QUARANTINE transition
- Verify Authorization token issuance and redemption round-trip latency

---

## Cost Model (10–50 nodes, cloud deployment)

| Component | Technology | Estimated Cost/month |
|---|---|---|
| Orchestrator cluster (3 nodes) | Kubernetes + Open Policy Agent | $200–$500 |
| Secrets management | HashiCorp Vault HCP | $0–$50 |
| Trust ledger | Apache Kafka (managed) | $100–$300 |
| SDN controller | OpenDaylight (OSS) | $0 |
| Honeypot layer | AWS GuardDuty / Honeyd | $50–$200 |
| RL engine (optional) | Ray RLlib (GPU spot) | $70–$360 |
| Node TPM chips (one-time) | TPM 2.0 per node | $2–$5/chip |
| **Total** | | **$500–$1,500/month** |

---

## Performance Overhead

| Operation | Overhead |
|---|---|
| QUIESCE claim (TPM attestation) | 50–100 ms (one-time per claim) |
| Authorization round-trip | 2–10 ms |
| IP/port rotation | 2–5 ms per event |
| eBPF enforcement per packet | < 1 µs |

---

© 2026 CHALLAYOGESWAR. Licensed under CC BY-NC-ND 4.0.
