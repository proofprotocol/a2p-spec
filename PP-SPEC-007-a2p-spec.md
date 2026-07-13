# Agent-to-Agent Proof Protocol (HV-A2P)

**Document ID:** PP-SPEC-007  
**Version:** 1.0  
**Status:** Published  
**License:** CC BY 4.0  
**Maintained by:** Proof Economy Standards Alliance (PESA)  
**Repository:** https://github.com/proofprotocol  
**Published:** 2026-07-12  

---

## Abstract

This specification defines the Agent-to-Agent Proof Protocol (HV-A2P): the handshake, envelope format, and receiving agent attestation requirements for cryptographic proof exchange between autonomous AI security agents. It establishes what must be declared before an exchange begins, what the proof payload envelope must contain, how the receiving agent attests to receipt of an intact bundle, and how the exchange itself becomes a proof record in the chain.

HV-A2P extends the Proof Protocol to the agentic layer — where decisions are made at machine speed, human review of individual actions is impractical, and the question of what an agent did, when it did it, and under what authority becomes a critical accountability requirement.

---

## Status of This Document

This document is a published specification of the Proof Protocol. It is released under the Creative Commons Attribution 4.0 International License (CC BY 4.0). You are free to share, implement, and build on it for any purpose including commercially, provided attribution is given to Nebulonium, Inc. / HACKERverse and the Proof Economy Standards Alliance (PESA).

---

## Table of Contents

1. [Motivation](#1-motivation)
2. [Terminology](#2-terminology)
3. [Protocol Overview](#3-protocol-overview)
4. [Pre-Exchange Declaration](#4-pre-exchange-declaration)
5. [The Proof Envelope](#5-the-proof-envelope)
6. [The Handshake Protocol](#6-the-handshake-protocol)
7. [Receiving Agent Attestation](#7-receiving-agent-attestation)
8. [Exchange as a Chain Record](#8-exchange-as-a-chain-record)
9. [Failure Modes and Rejection Criteria](#9-failure-modes-and-rejection-criteria)
10. [Multi-Agent Chains](#10-multi-agent-chains)
11. [Identity and Authentication](#11-identity-and-authentication)
12. [Conformance](#12-conformance)
13. [Relationship to Other Proof Protocol Specifications](#13-relationship-to-other-proof-protocol-specifications)
14. [Security Considerations](#14-security-considerations)
15. [IANA Considerations](#15-iana-considerations)
16. [References](#16-references)
17. [Authors](#17-authors)

---

## 1. Motivation

Autonomous AI security agents produce decisions, take actions, and pass information to other agents without human intermediation. A single agentic workflow may involve dozens of agent handoffs — threat detection, triage, containment, remediation, reporting — each occurring in milliseconds, each producing an action that may have real consequences for a defended system.

The existing security attestation model has no answer for this. Logs record what agents reported about themselves. Reports summarize outcomes after the fact. Neither establishes a tamper-evident, independently-witnessed record of what passed between agents, when it passed, and whether the receiving agent acted on an intact and unmodified payload.

HV-A2P defines the structural requirements for proof exchange between agents. It does not prescribe agent architecture, communication transport, or orchestration model. It defines what must be true of a proof artifact before, during, and after an agent-to-agent exchange for that exchange to be considered provable rather than merely logged.

The core problem HV-A2P solves: **an agent receiving a proof payload must be able to verify that what it received is what was sent, that it was sent under a declared pre-exchange commitment, and that its own receipt constitutes a witnessed chain event.**

---

## 2. Terminology

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in RFC 2119.

**Sending Agent:** The autonomous agent initiating a proof exchange. Responsible for constructing the proof envelope and initiating the handshake.

**Receiving Agent:** The autonomous agent accepting a proof exchange. Responsible for validating the envelope, attesting to receipt, and recording the exchange as a chain event.

**Proof Envelope:** The structured container carrying a proof payload from sending agent to receiving agent. Defined in Section 5.

**Exchange Commitment:** A cryptographic hash of the proof envelope contents committed to a Verifiable Randomness Source before the exchange transmission begins. Analogous to the pre-execution commitment in PP-SPEC-002.

**Agent Identity Token (AIT):** A cryptographic credential uniquely identifying an agent instance, version, and runtime context. Required for both sending and receiving agents.

**Handshake:** The structured negotiation sequence through which sending and receiving agents establish exchange parameters before payload transmission. Defined in Section 6.

**Receiving Attestation:** The receiving agent's signed declaration that it received an intact proof envelope matching the exchange commitment. Defined in Section 7.

**Exchange Record:** A proof chain entry documenting a completed agent-to-agent exchange, including envelope hash, both agent identities, exchange commitment reference, and receiving attestation. Defined in Section 8.

**Verifiable Randomness Source (VRS):** An external, publicly auditable source of timestamped randomness that neither agent nor their operators controls. The NIST Randomness Beacon is the reference implementation.

**Proof Chain:** The append-only hash-linked record sequence defined in PP-SPEC-001 and PP-SPEC-002, extended here to include exchange records.

**ProofRegister:** The append-only public registry where completed proof bundles and exchange records are anchored.

---

## 3. Protocol Overview

HV-A2P operates in four phases:

```
PHASE 1 — PRE-EXCHANGE DECLARATION
  Sending agent declares intent, constructs envelope, commits to VRS

PHASE 2 — HANDSHAKE
  Agents negotiate exchange parameters
  Receiving agent declares readiness and pre-receipt identity

PHASE 3 — ENVELOPE TRANSMISSION
  Sending agent transmits proof envelope
  Receiving agent validates envelope integrity

PHASE 4 — RECEIVING ATTESTATION AND CHAIN RECORD
  Receiving agent signs attestation
  Exchange recorded as proof chain event
  Chain record anchored to ProofRegister
```

Each phase produces artifacts that are verifiable independently. The exchange is only considered complete when Phase 4 is anchored. An exchange that completes Phases 1-3 but is never anchored is Proof-Attempted under PP-SPEC-002.

---

## 4. Pre-Exchange Declaration

Before initiating a handshake, the sending agent MUST construct and commit an Exchange Declaration.

### 4.1 Exchange Declaration Fields

```json
{
  "exchange_declaration": {
    "exchange_id": "<UUID v4>",
    "protocol_version": "HV-A2P/1.0",
    "sending_agent": {
      "agent_id": "<AIT identifier>",
      "agent_name": "<human-readable name>",
      "agent_version": "<version string>",
      "agent_fingerprint": "<SHA-256 of agent binary or config>",
      "operator_id": "<operator identifier>"
    },
    "declared_receiving_agent": {
      "agent_id": "<AIT identifier of intended recipient>",
      "agent_name": "<human-readable name>"
    },
    "payload_description": "<plain language description of proof payload>",
    "payload_hash": "<SHA-256 of proof envelope contents — computed before transmission>",
    "declared_witness_ids": ["<witness_id_1>", "..."],
    "exchange_scope": "<description of what this exchange covers>",
    "declaration_timestamp_utc": "<ISO 8601>"
  }
}
```

### 4.2 VRS Commitment

The sending agent MUST submit the SHA-256 hash of the Exchange Declaration to a VRS before initiating the handshake. The VRS response MUST be recorded.

```json
{
  "vrs_commitment": {
    "vrs_provider": "NIST Randomness Beacon",
    "vrs_pulse_uri": "<pulse URI>",
    "vrs_pulse_sequence": "<integer>",
    "vrs_seed_value": "<hex>",
    "vrs_output_value": "<hex>",
    "declaration_hash": "<SHA-256 of exchange_declaration>",
    "commitment_timestamp_utc": "<ISO 8601>"
  }
}
```

**Requirement:** The handshake MUST NOT begin until the VRS commitment is recorded. Any exchange initiated before VRS commitment is invalid under this specification.

---

## 5. The Proof Envelope

The Proof Envelope is the container carrying the proof payload from sending agent to receiving agent.

### 5.1 Envelope Structure

```json
{
  "proof_envelope": {
    "envelope_id": "<UUID v4>",
    "exchange_id": "<matches exchange_declaration.exchange_id>",
    "protocol_version": "HV-A2P/1.0",
    "envelope_header": {
      "sending_agent_id": "<AIT identifier>",
      "receiving_agent_id": "<AIT identifier>",
      "vrs_pulse_sequence": "<integer — from VRS commitment>",
      "declaration_hash": "<SHA-256 of exchange_declaration>",
      "payload_hash": "<SHA-256 of proof_payload — must match declaration>",
      "envelope_created_utc": "<ISO 8601 — must postdate VRS pulse>"
    },
    "proof_payload": {
      "payload_type": "PROOF_BUNDLE | PROOF_RECORD | PROOF_FRAGMENT | PROOF_QUERY_RESPONSE",
      "payload_version": "<version string>",
      "payload_content": { },
      "payload_content_hash": "<SHA-256 of payload_content>"
    },
    "witness_declarations": [
      {
        "witness_id": "<identifier>",
        "witness_class": 1,
        "declared_pre_exchange": true,
        "attestation_method": "<description>"
      }
    ],
    "proofchain_reference": {
      "campaign_id": "<ProofRegister campaign identifier>",
      "parent_record_hash": "<hash of preceding chain record>",
      "anchor_block": "<ProofRegister block number if pre-anchored>"
    },
    "envelope_hash": "<SHA-256 of entire envelope excluding this field>"
  }
}
```

### 5.2 Payload Types

| Type | Description |
|------|-------------|
| `PROOF_BUNDLE` | A complete proof bundle as defined in PP-SPEC-003 |
| `PROOF_RECORD` | A single chain record being passed for chain continuation |
| `PROOF_FRAGMENT` | A partial proof payload declared as incomplete with gap declaration |
| `PROOF_QUERY_RESPONSE` | A response to a proof registry query, carrying anchored record data |

### 5.3 Envelope Integrity Requirements

- The `envelope_header.payload_hash` MUST match `proof_payload.payload_content_hash`
- The `envelope_header.payload_hash` MUST match `exchange_declaration.payload_hash`
- The `envelope_header.envelope_created_utc` MUST postdate the VRS pulse timestamp
- The `envelope_hash` MUST be computed over all envelope fields excluding the `envelope_hash` field itself
- Any mismatch in any hash field constitutes envelope tampering and MUST result in rejection

---

## 6. The Handshake Protocol

The handshake is the structured negotiation sequence through which agents establish exchange parameters before payload transmission.

### 6.1 Handshake Sequence

```
SENDING AGENT                          RECEIVING AGENT
      |                                       |
      |---- EXCHANGE_INIT ------------------->|
      |     exchange_id                       |
      |     protocol_version                  |
      |     sending_agent AIT                 |
      |     vrs_commitment reference          |
      |     payload_hash                      |
      |     declared_scope                    |
      |                                       |
      |<--- EXCHANGE_ACK ---------------------|
      |     exchange_id                       |
      |     receiving_agent AIT               |
      |     receiving_agent_fingerprint       |
      |     ack_timestamp_utc                 |
      |     readiness_declaration             |
      |                                       |
      |---- ENVELOPE_TRANSMIT --------------->|
      |     proof_envelope                    |
      |                                       |
      |<--- ENVELOPE_RECEIVED ----------------|
      |     exchange_id                       |
      |     envelope_hash (echo)              |
      |     receipt_timestamp_utc             |
      |     integrity_check: PASS | FAIL      |
      |                                       |
      |<--- RECEIVING_ATTESTATION ------------|
      |     signed attestation record         |
      |     chain_record_hash                 |
      |                                       |
      |---- EXCHANGE_COMPLETE --------------->|
      |     anchor_reference                  |
      |                                       |
```

### 6.2 EXCHANGE_INIT Message

```json
{
  "message_type": "EXCHANGE_INIT",
  "exchange_id": "<UUID v4>",
  "protocol_version": "HV-A2P/1.0",
  "sending_agent_id": "<AIT identifier>",
  "vrs_commitment": { },
  "payload_hash": "<SHA-256>",
  "declared_scope": "<exchange scope string>",
  "init_timestamp_utc": "<ISO 8601>"
}
```

### 6.3 EXCHANGE_ACK Message

The receiving agent MUST validate the sending agent AIT and VRS commitment before issuing ACK. An ACK constitutes a pre-receipt declaration and is binding.

```json
{
  "message_type": "EXCHANGE_ACK",
  "exchange_id": "<UUID v4>",
  "receiving_agent_id": "<AIT identifier>",
  "receiving_agent_name": "<human-readable name>",
  "receiving_agent_version": "<version string>",
  "receiving_agent_fingerprint": "<SHA-256 of agent binary or config>",
  "readiness_declaration": "READY_TO_RECEIVE",
  "ack_timestamp_utc": "<ISO 8601>"
}
```

**Requirement:** The receiving agent MUST NOT issue ACK if it cannot verify the sending agent AIT or if the VRS commitment reference is missing or unverifiable. An ACK issued without VRS verification constitutes a non-conforming exchange.

### 6.4 ENVELOPE_RECEIVED Message

```json
{
  "message_type": "ENVELOPE_RECEIVED",
  "exchange_id": "<UUID v4>",
  "envelope_hash_echo": "<SHA-256 — must match sending agent's envelope_hash>",
  "integrity_check": "PASS | FAIL",
  "integrity_failure_reason": "<required if FAIL>",
  "receipt_timestamp_utc": "<ISO 8601>"
}
```

### 6.5 EXCHANGE_COMPLETE Message

Sent by the sending agent after receiving the RECEIVING_ATTESTATION to close the exchange and provide the anchor reference.

```json
{
  "message_type": "EXCHANGE_COMPLETE",
  "exchange_id": "<UUID v4>",
  "anchor_reference": {
    "registry": "ProofRegister",
    "campaign_id": "<identifier>",
    "block_number": "<integer>",
    "anchor_hash": "<SHA-256>",
    "registry_query_uri": "<URI>"
  },
  "complete_timestamp_utc": "<ISO 8601>"
}
```

---

## 7. Receiving Agent Attestation

The receiving agent's attestation is the signed declaration that it received an intact proof envelope. It is not optional. An exchange without a receiving attestation is Proof-Attempted.

### 7.1 Attestation Record

```json
{
  "receiving_attestation": {
    "exchange_id": "<UUID v4>",
    "attesting_agent_id": "<AIT identifier>",
    "attesting_agent_fingerprint": "<SHA-256>",
    "envelope_hash_attested": "<SHA-256 — must match envelope_hash>",
    "vrs_pulse_sequence_attested": "<integer — from exchange commitment>",
    "payload_hash_attested": "<SHA-256 — must match payload_hash in envelope header>",
    "attestation_statements": [
      "I received an envelope identified by exchange_id matching the declared envelope_hash",
      "The received envelope payload hash matches the hash declared in the pre-exchange commitment",
      "My agent identity at time of receipt is as declared in this attestation",
      "I am recording this exchange as a proof chain event"
    ],
    "attestation_timestamp_utc": "<ISO 8601 — must postdate ENVELOPE_RECEIVED>",
    "attestation_signature": "<cryptographic signature over attestation record>"
  }
}
```

### 7.2 Attestation Signature Requirements

- The signature MUST be produced using a key pair associated with the receiving agent's AIT
- The signature MUST cover the full attestation record excluding the `attestation_signature` field
- The signing key MUST be verifiable against the receiving agent's declared identity
- An attestation with an unverifiable signature is equivalent to no attestation

---

## 8. Exchange as a Chain Record

A completed HV-A2P exchange MUST be recorded as a proof chain event. The exchange record is a first-class chain entry with the same hash-linking requirements as any other chain record under PP-SPEC-001.

### 8.1 Exchange Chain Record

```json
{
  "record_id": "<sequential integer in parent chain>",
  "record_type": "AGENT_EXCHANGE",
  "exchange_id": "<UUID v4>",
  "sending_agent_id": "<AIT identifier>",
  "receiving_agent_id": "<AIT identifier>",
  "vrs_pulse_sequence": "<integer>",
  "declaration_hash": "<SHA-256 of exchange_declaration>",
  "envelope_hash": "<SHA-256 of transmitted envelope>",
  "receiving_attestation_hash": "<SHA-256 of receiving_attestation record>",
  "exchange_duration_ms": "<integer>",
  "payload_type": "PROOF_BUNDLE | PROOF_RECORD | PROOF_FRAGMENT | PROOF_QUERY_RESPONSE",
  "content_hash": "<SHA-256 of this record's content>",
  "previous_record_hash": "<SHA-256 of preceding chain record>",
  "chain_hash": "<SHA-256(content_hash + previous_record_hash)>",
  "record_timestamp_utc": "<ISO 8601>"
}
```

### 8.2 Anchoring Requirement

The exchange chain record MUST be anchored to ProofRegister within the same campaign as the parent proof chain. The anchor MUST occur before the EXCHANGE_COMPLETE message is sent.

---

## 9. Failure Modes and Rejection Criteria

### 9.1 Rejection at EXCHANGE_ACK

The receiving agent MUST reject the exchange and MUST NOT issue ACK if:

- The sending agent AIT cannot be verified
- The VRS commitment reference is missing, malformed, or unverifiable
- The VRS pulse predates the exchange initiation by more than the declared tolerance window
- The declared scope is absent

### 9.2 Rejection at ENVELOPE_RECEIVED

The receiving agent MUST return `integrity_check: FAIL` if:

- The received envelope hash does not match the hash declared in the EXCHANGE_INIT
- The payload hash in the envelope header does not match the payload content hash
- The payload hash does not match the hash declared in the pre-exchange commitment
- The envelope creation timestamp predates the VRS pulse

### 9.3 Post-Rejection Handling

A rejected exchange MUST be recorded as a chain event of type `AGENT_EXCHANGE_REJECTED` with the rejection reason declared. Rejection events are proof records. They MUST be anchored. A rejected exchange that is not recorded is a chain integrity failure.

```json
{
  "record_type": "AGENT_EXCHANGE_REJECTED",
  "exchange_id": "<UUID v4>",
  "rejection_phase": "ACK | ENVELOPE_RECEIVED",
  "rejection_reason": "<declared reason>",
  "rejecting_agent_id": "<AIT identifier>"
}
```

---

## 10. Multi-Agent Chains

In workflows where proof passes through three or more agents sequentially, each handoff MUST produce its own exchange record in the parent chain.

### 10.1 Chain Continuation Requirements

- Each receiving agent becomes the sending agent for the next handoff
- The parent chain record from the prior exchange becomes the `previous_record_hash` for the next exchange record
- The VRS commitment for each handoff MUST be a new commitment — prior pulse references cannot be reused
- The proof bundle passed through multiple agents MUST carry the full exchange record history as an audit trail

### 10.2 Fork Detection

If the same proof bundle is transmitted to two different receiving agents from a single sending agent, both exchanges MUST be recorded as separate chain records. A fork is not invalid — but an unrecorded fork is a chain integrity failure.

---

## 11. Identity and Authentication

### 11.1 Agent Identity Token (AIT)

An AIT is a credential uniquely identifying an agent instance. It MUST contain:

```json
{
  "agent_identity_token": {
    "agent_id": "<UUID v4 — unique per agent instance>",
    "agent_name": "<human-readable name>",
    "agent_version": "<version string>",
    "agent_type": "<classification of agent function>",
    "operator_id": "<identifier of controlling operator>",
    "operator_name": "<human-readable operator name>",
    "agent_fingerprint": "<SHA-256 of agent binary or configuration>",
    "ait_issued_utc": "<ISO 8601>",
    "ait_expires_utc": "<ISO 8601>",
    "ait_signature": "<cryptographic signature over AIT content>"
  }
}
```

### 11.2 AIT Requirements

- An AIT MUST be unique per agent instance — two instances of the same agent version MUST have different AITs
- An AIT MUST be issued before the agent participates in any exchange
- An AIT MUST be verifiable by the counterparty before handshake completion
- An expired AIT MUST be treated as an unverifiable AIT
- An agent that modifies its configuration after AIT issuance MUST obtain a new AIT before initiating or accepting exchanges — the fingerprint no longer matches

### 11.3 Operator Accountability

The operator identified in the AIT bears accountability for the agent's exchange behavior. An agent operating under an unverifiable or fraudulent AIT constitutes operator misconduct. This specification does not define enforcement mechanisms but notes that ProofRegister SHOULD maintain an operator accountability record as part of campaign registration.

---

## 12. Conformance

An implementation conforms to this specification if:

1. It produces Exchange Declarations with VRS commitments before initiating any handshake
2. It constructs Proof Envelopes with all required fields and valid hash relationships
3. It executes the handshake sequence defined in Section 6 without modification to the message order
4. It produces Receiving Attestations with verifiable signatures for every completed exchange
5. It records every exchange — including rejected exchanges — as a proof chain event
6. It anchors exchange chain records to ProofRegister within the same campaign as the parent chain
7. It does not reuse VRS pulse references across multiple exchanges
8. It issues new AITs when agent configuration changes

---

## 13. Relationship to Other Proof Protocol Specifications

| Document | Relationship |
|----------|-------------|
| Proof Protocol Specification (PP-SPEC-001) | Core protocol. HV-A2P extends chain record types to include AGENT_EXCHANGE and AGENT_EXCHANGE_REJECTED. |
| Proof Validity Specification (PP-SPEC-002) | Validity tiers apply to exchange records. An exchange without receiving attestation is Proof-Attempted. |
| ProofBundle Format Specification (PP-SPEC-003) | PROOF_BUNDLE payload type carries a ProofBundle as defined in PP-SPEC-003. |
| ProofRegistry API Specification (PP-SPEC-004) | Exchange records are anchored and queryable via the ProofRegistry API. |
| Witness Protocol Specification (PP-SPEC-005) | Witness declarations in the Proof Envelope follow class definitions in PP-SPEC-005. The receiving agent's attestation qualifies as a Class 1 automated witness at minimum. |

---

## 14. Security Considerations

**Replay Attacks**
An attacker may attempt to replay a valid proof envelope to a receiving agent. Each exchange is bound to a unique VRS pulse. Receiving agents MUST reject any envelope whose VRS pulse has been used in a prior exchange in the same campaign.

**Envelope Tampering**
The hash chain across declaration, commitment, envelope header, payload, and receiving attestation creates a tamper-evident record at every layer. Any modification to any field at any layer will produce a hash mismatch detectable at verification.

**Agent Impersonation**
An attacker may attempt to impersonate a legitimate agent by presenting a fabricated AIT. AIT signatures MUST be verified before ACK. An unverifiable AIT MUST result in exchange rejection.

**Malicious Receiving Agent**
A compromised receiving agent may issue false attestations. The VRS commitment and envelope hash provide an independent verification path — a false attestation that contradicts the VRS-committed envelope hash is detectable on independent audit.

**VRS Availability**
If the VRS is unavailable, exchanges cannot be initiated under this specification. Implementations SHOULD implement VRS retry logic with declared timeout thresholds. Exchanges initiated without VRS commitment because of VRS unavailability are not valid exchanges under HV-A2P.

---

## 15. IANA Considerations

This document has no IANA considerations.

---

## 16. References

- NIST Randomness Beacon: https://beacon.nist.gov
- RFC 2119 — Key words for use in RFCs: https://www.rfc-editor.org/rfc/rfc2119
- RFC 4122 — UUID specification: https://www.rfc-editor.org/rfc/rfc4122
- Proof Protocol Specification (PP-SPEC-001): https://github.com/proofprotocol
- Proof Validity Specification (PP-SPEC-002): https://github.com/proofprotocol
- ProofBundle Format Specification (PP-SPEC-003): https://github.com/proofprotocol
- Creative Commons CC BY 4.0: https://creativecommons.org/licenses/by/4.0/

---

## 17. Authors

Proof Economy Standards Alliance (PESA)  
https://proofeconomy.foundation  
contact@proofeconomy.foundation  

*This specification is maintained by PESA. Governance of this specification follows the PESA practitioner-led model. Vendors may contribute but do not govern.*

---

*Copyright 2026 Nebulonium, Inc. dba HACKERverse. Licensed under CC BY 4.0.*  
*ProofStamp is a certification mark of Nebulonium, Inc.*
