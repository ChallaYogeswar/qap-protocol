# QAP Protocol Specification — Version 1.0

**Author:** CHALLAYOGESWAR (challayogeswar8790@gmail.com)
**Version:** 1.0
**Date:** 21 May 2026
**Status:** Research Proposal
**License:** CC BY-NC-ND 4.0

---

## 1. Purpose and Scope

This document defines the complete technical specification for the
Quiesce-Authorize Protocol (QAP), including node state definitions,
state transition rules, Authorization token format, Orchestrator
responsibilities, quorum enforcement, and trust score computation.

---

## 2. Node States

Every node registered in a QAP-governed network operates in exactly one
of the following three states at any given time:

| State | Identifier | Description |
|---|---|---|
| Active | `ACTIVE` | Node is fully operational. Participates in traffic routing. Eligible to issue Authorization tokens to QUIESCE nodes. Participates in IP/port rotation. |
| Quiesce | `QUIESCE` | Node has claimed a temporary protected rest state, validated by a TPM 2.0 hardware-attested load certificate. Not eligible to issue tokens. Subject to TTL expiry. |
| Quarantine | `QUARANTINE` | Node has been flagged as compromised, or has exceeded QUIESCE TTL without receiving a valid Authorization. All rights suspended pending administrative review. |

### 2.1 State Transition Rules

```
ACTIVE    → QUIESCE      : Valid TPM-attested load certificate submitted to Orchestrator,
                           quorum constraint satisfied
QUIESCE   → ACTIVE       : Valid Authorization JWT received and verified by Orchestrator
QUIESCE   → QUARANTINE   : TTL expires without valid Authorization received
ACTIVE    → QUARANTINE   : Node flagged as compromised by threat detection subsystem
QUARANTINE → ACTIVE      : Administrative reinstatement after forensic review
```

---

## 3. The QUIESCE Mechanism

### 3.1 Claim Process

A node wishing to enter QUIESCE state MUST:

1. Generate a load certificate containing:
   - Current CPU utilization (percentage)
   - Current memory pressure (percentage)
   - Current bandwidth saturation (percentage)
   - Node identity (public key fingerprint)
   - Timestamp (UTC, RFC 3339)
   - Nonce (128-bit random value)
2. Sign the load certificate using the node's TPM 2.0 attestation key
3. Submit the signed certificate to the Orchestrator via the mTLS-authenticated API

### 3.2 Orchestrator Validation

On receipt of a QUIESCE claim, the Orchestrator MUST:

1. Verify the TPM attestation signature against the node's registered endorsement key
2. Verify the timestamp is within a 30-second window of server time
3. Verify the nonce has not been previously submitted
4. Evaluate the quorum constraint (see Section 5)
5. Compare attested telemetry against observed behavioral signals
6. If all checks pass: transition node to QUIESCE state and start TTL timer
7. If any check fails: reject the claim and apply a trust score penalty (see Section 6)

### 3.3 QUIESCE TTL

- Default TTL: **30 seconds** (configurable per deployment, minimum 10 seconds)
- TTL begins at the moment the Orchestrator grants QUIESCE state
- If the node has not received a valid Authorization within the TTL window, the
  Orchestrator MUST automatically transition the node to QUARANTINE

---

## 4. The AUTHORIZATION Mechanism

### 4.1 Token Format

An Authorization token is a JSON Web Token (JWT) with the following claims:

```json
{
  "iss": "<orchestrator-node-id>",
  "sub": "<target-node-id>",
  "iat": "<issued-at-unix-timestamp>",
  "exp": "<expiry-unix-timestamp (iat + 30)>",
  "jti": "<unique-nonce-128-bit-hex>",
  "issuing_node": "<issuing-peer-node-id>",
  "issuing_node_trust_score": "<score-at-time-of-issuance>",
  "target_node": "<target-node-id>",
  "qap_version": "1.0"
}
```

The token MUST be signed by the Orchestrator's private signing key (RS256 minimum,
ES256 recommended). The issuing peer node requests the token from the Orchestrator;
the Orchestrator signs it only if the issuing peer's trust score meets the threshold.

### 4.2 Issuance Eligibility

A peer node MAY request Authorization issuance for a QUIESCE node ONLY IF:

- The requesting node is currently in ACTIVE state
- The requesting node's trust score is ≥ the configured threshold (default: 70/100)
- The requesting node is not the same node as the QUIESCE target
- The requesting node has not exceeded the rate limit for Authorization requests
  (default: 3 requests per 60-second window)

### 4.3 Redemption Process

1. The ACTIVE peer requests an Authorization token from the Orchestrator for the
   target QUIESCE node
2. The Orchestrator validates eligibility (Section 4.2) and signs the token
3. The ACTIVE peer delivers the token to the QUIESCE node via a direct
   mTLS-authenticated channel
4. The QUIESCE node presents the token to the Orchestrator
5. The Orchestrator verifies: signature, expiry, nonce uniqueness, issuing node
   identity, target node identity
6. On successful verification: node transitions from QUIESCE to ACTIVE;
   nonce is recorded in the 30-second nonce registry to prevent replay

---

## 5. Quorum Constraint

The Orchestrator MUST maintain the following invariant at all times:

> **At least 51% of all registered ACTIVE and QUIESCE nodes must be in ACTIVE
> state.**

A QUIESCE claim that would violate this constraint MUST be rejected by the
Orchestrator. The requesting node remains in ACTIVE state and receives a
`QUORUM_VIOLATION` rejection code.

Nodes in QUARANTINE state do not count toward the quorum calculation.

### 5.1 Distributed Quorum Under Orchestrator Clustering

In a clustered Orchestrator deployment (minimum 3 nodes, Raft consensus), quorum
state is replicated in real time across all Orchestrator instances. Any instance
may service QUIESCE claims and Authorization requests. Quorum calculations use the
replicated state, not local state.

---

## 6. Trust Score System

### 6.1 Score Range and Threshold

- Scores range from **0 to 100**
- New nodes are initialized at **score 60**
- Authorization issuance threshold (default): **70**
- Scores are recomputed by the Orchestrator on each relevant event

### 6.2 Score Modifiers

| Event | Score Impact |
|---|---|
| Valid QUIESCE claim granted | −0 (neutral) |
| QUIESCE claim rejected (invalid cert) | −5 |
| QUIESCE claim rejected (quorum violation) | −1 |
| Repeated QUIESCE claims within 5 minutes | −3 per repeat |
| Attested telemetry inconsistent with observed behavior | −10 |
| Authorization issued; recipient resumed normally | +2 |
| QUARANTINE event | −25 |
| Each 24-hour period in good ACTIVE standing | +1 (max +10 recovery) |

### 6.3 Score Ledger

All score modification events MUST be recorded as immutable, Orchestrator-signed
entries in the trust ledger (Section 7). Scores MUST NOT be modified outside the
defined event model.

---

## 7. Trust Ledger

The trust ledger is an append-only distributed event log. Every QUIESCE claim,
Authorization issuance, Authorization redemption, QUARANTINE event, and trust
score modification MUST be recorded as a signed ledger entry.

### 7.1 Entry Format

```json
{
  "event_id": "<uuid>",
  "event_type": "<QUIESCE_CLAIM | AUTH_ISSUED | AUTH_REDEEMED | QUARANTINE | SCORE_MODIFIED>",
  "node_id": "<affected-node-id>",
  "timestamp": "<UTC RFC 3339>",
  "details": { },
  "orchestrator_signature": "<base64-encoded RS256 signature>"
}
```

### 7.2 Implementation

The trust ledger SHOULD be implemented on Apache Kafka with log compaction
disabled (retain all events). The ledger MUST be replicated across all
Orchestrator instances. Read access is available to authorized administrative
principals via a read-only authenticated API.

---

## 8. MTD Engine

All ACTIVE nodes MUST participate in continuous network property rotation:

| Property | Rotation Interval | Mechanism |
|---|---|---|
| IP address | 15–60 seconds (configurable) | SDN controller (OpenDaylight / ONOS) |
| Listening port | 15–60 seconds (configurable) | SDN controller |
| Protocol dialect | Per-session | MPD-style PRNG + pre-shared secret |

Rotation commands are issued by the Orchestrator's MTD engine to the SDN
controller. All participating nodes receive updated routing tables automatically.

QUIESCE and QUARANTINE nodes do not participate in rotation during their
respective states.

---

## 9. eBPF Enforcement

QUIESCE and QUARANTINE state restrictions MUST be enforced at the Linux kernel
level via eBPF programs loaded by the Orchestrator agent running on each node.

eBPF programs enforce:
- Traffic restriction for QUIESCE nodes (inbound/outbound limited to Orchestrator
  API and Authorization receipt channels only)
- Full traffic isolation for QUARANTINE nodes (Orchestrator API only)
- Prevention of user-space bypass of state restrictions

---

## 10. Orchestrator API

The Orchestrator exposes a mTLS-authenticated REST API. All connections MUST
use TLS 1.3 minimum with mutual certificate authentication.

| Endpoint | Method | Description |
|---|---|---|
| `/v1/quiesce/claim` | POST | Submit a QUIESCE claim with load certificate |
| `/v1/quiesce/authorize` | POST | Request Authorization token for a QUIESCE node |
| `/v1/quiesce/redeem` | POST | Redeem an Authorization token |
| `/v1/node/state/{node_id}` | GET | Query current state of a node |
| `/v1/node/trust/{node_id}` | GET | Query trust score of a node |
| `/v1/ledger/events` | GET | Query trust ledger (admin only) |
| `/v1/orchestrator/health` | GET | Orchestrator health check |

---

## 11. Decoy Honeypot Integration

The Orchestrator MUST maintain a configurable pool of decoy nodes. Decoy nodes:

- Present a behavioral profile indistinguishable from genuine isolated ACTIVE nodes
- Participate in normal IP/port rotation
- Do NOT hold real Authorization-issuance rights
- Log all interaction attempts as threat intelligence events
- Feed captured attacker signatures to the RL engine (if enabled)

Any node that interacts with a decoy node MUST be immediately flagged and subject
to QUARANTINE initiation.

---

## 12. Changelog

| Version | Date | Description |
|---|---|---|
| 1.0 | May 2026 | Initial specification release |

---

© 2026 CHALLAYOGESWAR (challayogeswar8790@gmail.com). All rights reserved.
Licensed under CC BY-NC-ND 4.0. See [LICENSE](../LICENSE).
