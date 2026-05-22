# QAP — Quiesce-Authorize Protocol

> **A Peer-Authenticated Dynamic Defense Architecture for Zero-Trust Network Security**

[![License: CC BY-NC-ND 4.0](https://img.shields.io/badge/License-CC%20BY--NC--ND%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-nd/4.0/)
[![Status: Research Proposal](https://img.shields.io/badge/Status-Research%20Proposal-blue.svg)]()
[![Version](https://img.shields.io/badge/Version-1.0-green.svg)]()
[![Author](https://img.shields.io/badge/Author-CHALLAYOGESWAR-navy.svg)]()

---

## Overview

**QAP (Quiesce-Authorize Protocol)** is a novel peer-authenticated dynamic defense framework that introduces a stateful, trust-scored session negotiation model for enterprise network security.

In QAP, network nodes dynamically advertise protected rest states called **Quiesce states** and resume active participation through cryptographically signed peer-issued tokens called **Authorizations**. The protocol enforces:

- Mutual exclusion constraints on Quiesce claims
- Quorum-based availability guarantees across the network
- Hardware-attested load certification via TPM 2.0
- Trust-scored peer Authorization issuance
- Kernel-level enforcement via eBPF

QAP is designed as a **complementary layer** to existing Zero-Trust architectures, Software Defined Networking (SDN) controllers, and Moving Target Defense (MTD) frameworks — not a replacement for them.

---

## The Problem

Modern enterprise networks operate under a fundamentally asymmetric threat model:

- Attackers conduct prolonged reconnaissance against **static infrastructure**
- Defenders are constrained to **reactive postures** — responding only after detection
- Zero-trust frameworks address identity and access control but provide **no mechanism** for nodes to dynamically negotiate protected states under active threat conditions

QAP addresses this gap directly.

---

## Core Mechanics

| Component | Description |
|---|---|
| **Quiesce state** | A node claims a temporary, hardware-attested protected rest state |
| **Authorization token** | A trust-scored peer issues a signed JWT to revive a Quiesced node |
| **Orchestrator** | Central control-plane authority; manages state, signing keys, quorum, and trust scores |
| **Mutual exclusion** | Only one node per policy group may hold Quiesce state at a time |
| **Quorum rule** | ≥ 51% of nodes must remain ACTIVE at all times |
| **Trust ledger** | Append-only distributed log of all Quiesce/Authorization events |
| **MTD engine** | Continuous IP, port, and protocol rotation invalidates attacker reconnaissance |

---

## Node State Machine

```
         ┌─────────────────────────────────────────────┐
         │                                             │
    ┌────▼────┐   Quiesce claim    ┌──────────┐        │
    │ ACTIVE  │ ─────────────────► │ QUIESCE  │        │
    │         │                    │          │        │
    │         │ ◄───────────────── │  (TTL)   │        │
    └────┬────┘  Authorization JWT └─────┬────┘        │
         │                              │              │
         │ compromised / TTL expired    │ TTL expired  │
         ▼                              ▼              │
    ┌──────────────────────────────────────────┐       │
    │              QUARANTINE                  │       │
    │   (stripped of all rights, admin review) │       │
    └──────────────────────────────────────────┘       │
                           │                           │
                           └───── reinstated ──────────┘
```

---

## What Makes QAP Different

Comparison against existing approaches:

| Capability | Standard MTD | Zero-Trust | MTD + Game Theory | **QAP** |
|---|:---:|:---:|:---:|:---:|
| IP / port rotation | ✅ | ❌ | ✅ | ✅ |
| Node rest-state negotiation | ❌ | ❌ | ❌ | ✅ |
| Peer Authorization revival | ❌ | ❌ | ❌ | ✅ |
| Hardware load attestation (TPM 2.0) | ❌ | ⚠️ | ❌ | ✅ |
| Mutual exclusion on rest states | ❌ | ❌ | ❌ | ✅ |
| Quorum availability guarantee | ❌ | ❌ | ❌ | ✅ |
| Trust-scored Authorization issuance | ❌ | ⚠️ | ❌ | ✅ |
| RL adaptive parameter tuning | ⚠️ | ❌ | ✅ | ✅ |
| eBPF kernel-level enforcement | ❌ | ❌ | ❌ | ✅ |
| Immutable audit ledger | ❌ | ⚠️ | ❌ | ✅ |

---

## Repository Structure

```
qap-protocol/
├── README.md                    ← This file
├── LICENSE                      ← CC BY-NC-ND 4.0
├── SECURITY.md                  ← Responsible disclosure policy
├── CONTRIBUTING.md              ← Contribution guidelines
├── CITATION.cff                 ← How to cite this work
│
├── spec/
│   ├── QAP-SPEC-v1.0.md         ← Full protocol specification
│   ├── threat-model.md          ← Threat model and attack surface analysis
│   └── state-machine.md         ← Formal node state machine definition
│
├── docs/
│   ├── whitepaper.md            ← Full research whitepaper (Markdown)
│   ├── architecture.md          ← System architecture and integration guide
│   ├── deployment.md            ← Deployment guide and cost model
│   └── faq.md                   ← Frequently asked questions
│
├── diagrams/
│   ├── state-machine.svg        ← Node state machine diagram
│   ├── architecture.svg         ← Full system architecture diagram
│   └── threat-model.svg         ← Threat model diagram
│
└── .github/
    ├── ISSUE_TEMPLATE/
    │   ├── bug_report.md
    │   └── research_feedback.md
    └── workflows/
        └── validate-spec.yml
```

---

## Threat Coverage

QAP directly addresses the following STRIDE threat categories:

- **Reconnaissance & lateral movement** — MTD engine continuously rotates IP, port, and protocol dialect
- **Denial of service via mass Quiesce** — Quorum constraint rejects claims that drop ACTIVE nodes below 51%
- **Evasion via simulated exhaustion** — TPM 2.0 hardware attestation makes load certificate forgery computationally infeasible
- **Privilege escalation on compromise** — Compromised nodes enter QUARANTINE immediately, stripped of all rights
- **Replay attacks on Authorization tokens** — Per-token nonce with 30-second registry; Orchestrator rejects all replays

---

## Integration Stack

QAP integrates with:

- **SDN** (OpenDaylight / ONOS) — executes IP and port rotation
- **HashiCorp Vault** — manages Orchestrator signing keys
- **Google BeyondCorp / Zero-Trust Proxy** — Quiesce/ACTIVE state surfaced as posture signal
- **Apache Kafka** — append-only SHOCK history ledger
- **AWS GuardDuty / Honeyd** — honeypot infrastructure and threat intelligence
- **TPM 2.0** — hardware-rooted load attestation
- **eBPF** — kernel-level state enforcement

---

## Estimated Deployment Cost (10–50 nodes)

| Component | Technology | Cost/month |
|---|---|---|
| Orchestrator cluster (3 nodes) | Kubernetes + OPA | $200–$500 |
| Secrets management | HashiCorp Vault | $0–$50 |
| Event ledger | Apache Kafka | $100–$300 |
| SDN controller | OpenDaylight (OSS) | $0 |
| Honeypot layer | AWS GuardDuty | $50–$200 |
| RL engine (optional) | Ray RLlib | $70–$360 |
| **Total** | | **$500–$1,500** |

---

## Author

**CHALLAYOGESWAR**
📧 challayogeswar8790@gmail.com

---

## License

This work is licensed under **Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0)**.

You may read, share, and cite this work with attribution. You may **not** use it commercially, modify it, or republish it without explicit written permission from the author.

Full license text: [LICENSE](./LICENSE)

---

## Citation

If you reference this work in research or publications, please cite as:

```
CHALLAYOGESWAR. (2026). QAP — Quiesce-Authorize Protocol: A Peer-Authenticated
Dynamic Defense Architecture for Zero-Trust Network Security (Version 1.0).
GitHub. https://github.com/CHALLAYOGESWAR/qap-protocol
```

Or use the [CITATION.cff](./CITATION.cff) file for automated citation tools.

---

## Disclaimer

All protocol designs, architectural specifications, threat models, and enhancement proposals in this repository are the original intellectual work of the author. This repository serves as a timestamped public record of prior art. All rights reserved under the terms of the LICENSE file.
