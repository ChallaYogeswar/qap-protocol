# QAP Threat Model

**Author:** CHALLAYOGESWAR (challayogeswar8790@gmail.com)
**Version:** 1.0 | **Date:** 21 May 2026

---

## Threat Categories and QAP Mitigations

| # | Threat | STRIDE Class | Attack Scenario | QAP Mitigation |
|---|---|---|---|---|
| T1 | Reconnaissance & lateral movement | Information Disclosure | Attacker scans for open ports and stable IPs to build a network map before exploiting | MTD engine rotates IP, port, and protocol dialect every 15–60 seconds; attacker map is continuously invalidated |
| T2 | Evasion via simulated exhaustion | Spoofing | Compromised node fakes high CPU/memory load to enter QUIESCE and avoid inspection | TPM 2.0 hardware attestation; certificate forgery is computationally infeasible; inconsistent telemetry triggers trust score penalty |
| T3 | Coordinated denial of service via mass QUIESCE | Denial of Service | Adversary induces multiple nodes to claim QUIESCE simultaneously, collapsing availability | Quorum constraint: Orchestrator rejects QUIESCE claims that drop ACTIVE nodes below 51% |
| T4 | Unauthenticated Authorization issuance | Elevation of Privilege | Compromised node issues a revival token to a QUIESCE peer, restoring it to ACTIVE under attacker control | Authorization tokens are Orchestrator-signed JWTs; issuing node must hold trust score ≥ 70; compromised nodes are quarantined before they can issue |
| T5 | Privilege escalation on compromise | Elevation of Privilege | Compromised node immediately pivots and acts as internal attacker | Compromised nodes enter QUARANTINE instantly; all Key-issuance and QUIESCE rights stripped; no data-plane access |
| T6 | Replay attacks on Authorization tokens | Tampering | Intercepted Authorization JWT reused to revive additional nodes | Per-token nonce with 30-second registry; Orchestrator rejects any previously presented nonce |
| T7 | Deterministic last-node targeting | Denial of Service / Tampering | Attacker identifies the single remaining ACTIVE node as a guaranteed escalation target | Decoy honeypot nodes mimic isolated ACTIVE nodes; attacker is drawn to traps; real isolated node identity is obscured |
| T8 | Orchestrator single-point-of-failure | Denial of Service | Attacker compromises or DoSes the Orchestrator, disabling all QAP guarantees | Distributed Orchestrator cluster (minimum 3 nodes, Raft consensus); no single instance is authoritative |
| T9 | QUIESCE state as indefinite hiding | Denial of Service | Node stays in QUIESCE indefinitely to avoid monitoring and scanning | QUIESCE TTL (30 seconds default); expiry without valid Authorization triggers automatic QUARANTINE |
| T10 | Low-trust node issuing Authorizations | Elevation of Privilege | Newly joined or recently reinstated node issues Authorizations to revive compromised peers | Trust score threshold enforced (default: 70/100); new nodes initialized at 60 and must earn Authorization rights |

---

## Out-of-Scope Threats (Current Version)

The following threats are acknowledged but not addressed by QAP v1.0:

- **Physical hardware compromise** of a TPM chip — QAP assumes physical security of enrolled nodes
- **Orchestrator insider threat** — addressed by external audit of Orchestrator key management (HashiCorp Vault) and the trust ledger, but not by the protocol itself
- **Sub-10ms latency requirements** — Key negotiation adds 2–10ms; QAP is not suitable for hard real-time systems

---

© 2026 CHALLAYOGESWAR. Licensed under CC BY-NC-ND 4.0.
