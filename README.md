> **Zenodo DOI:** [10.5281/zenodo.21379780](https://doi.org/10.5281/zenodo.21379780) — Published 2026-07-15

# PP-SPEC-007 · Agent-to-Agent Proof Protocol™ (PP-A2P™)

**Document ID:** PP-SPEC-007  
**Version:** 1.0  
**Status:** Published  
**License:** CC BY 4.0  
**Maintained by:** Proof Economy™ Standards Alliance (PESA)  
**Repository:** https://github.com/proofprotocol/a2p-spec  
**Published:** 2026-07-12  

---

## Abstract

This specification defines the HACKERverse Agent-to-Agent Proof Protocol™ (PP-A2P™): the trust signaling mechanism by which autonomous agents verify behavioral proof before accepting instructions from, transacting with, or delegating authority to other agents.

In the quint economy - where agents transact with agents at machine speed across enterprise systems - trust cannot depend on human review. It must be cryptographic, verifiable, and anchored to an independent public record.

Agents and humans do not trust agents. They trust proof.

PP-A2P™ defines how that proof is requested, presented, verified, and recorded between autonomous agents.

---

## Status of This Document

This document is a published specification of the Proof Protocol™. It is released under the Creative Commons Attribution 4.0 International License (CC BY 4.0). You are free to share, implement, and build on it for any purpose including commercially, provided attribution is given to Nebulonium, Inc. / HACKERverse and the Proof Economy™ Standards Alliance (PESA).

---

## Table of Contents

1. [Motivation](#1-motivation)
2. [Terminology](#2-terminology)
3. [Trust Tiers](#3-trust-tiers)
4. [Proof Request](#4-proof-request)
5. [Proof Presentation](#5-proof-presentation)
6. [Verification Procedure](#6-verification-procedure)
7. [ProofRegister™ Integration](#7-proofregister-integration)
8. [ProofWitness™ Witness Layer](#8-proofwitness-witness-layer)
9. [Conformance](#9-conformance)
10. [References](#10-references)
11. [Authors](#11-authors)

---

## 1. Motivation

Autonomous agents are booking travel, executing trades, writing code, sending emails, and making purchasing decisions on behalf of humans and other agents. When an agent receives an instruction from another agent, it has no way to verify that the instructing agent behaved safely in the past, is operating under a known policy, or has not been compromised.

The result is a trust vacuum at the core of the agentic economy. Agents delegate to agents that delegate to agents, with no verifiable record of behavior at any layer.

PP-A2P™ fills that vacuum. It defines a lightweight proof handshake that any agent can implement to request, present, and verify behavioral proof before accepting an instruction or completing a transaction.

---

## 2. Terminology

**Requesting Agent** - the agent requesting proof from a counterparty before proceeding.

**Presenting Agent** - the agent presenting proof of its own behavioral history.

**ProofBundle™** - the portable evidence artifact defined in PP-SPEC-003.

**ProofRegister™ Record** - a permanent anchored record in ProofRegister™ identified by a Proof Record ID (PR-YYYY-NNNNN).

**ProofStamp™ Token** - the certification mark authorization token issued by HACKERverse attesting that a product or agent meets Proof Protocol™ certification criteria.

**ProofWitness™** - the shadow attestation layer that witnesses agent runtime behavior and assembles ProofBundles for anchoring.

**Trust Tier** - the level of proof required before an agent proceeds with an interaction. Defined in Section 3.

---

## 3. Trust Tiers

PP-A2P™ defines three trust tiers. The requesting agent declares the minimum tier required before proceeding.

| Tier | Name | Requirement |
|------|------|-------------|
| T1 | Self-Asserted | Agent presents a signed identity claim. No proof required. Lowest trust. |
| T2 | Registry-Verified | Agent presents a valid ProofRegister™ Record ID. Requesting agent verifies the record exists and is not revoked. |
| T3 | Stamp-Certified | Agent presents a valid ProofStamp™ Token. Requesting agent verifies the token against the HACKERverse public key. Highest trust. |

T1 is the SAO tier - self-attesting, institutionally incomplete. T2 and T3 require independent verification. Agents operating in regulated environments or high-stakes workflows should require T3.

---

## 4. Proof Request

A requesting agent initiates a proof handshake by sending a ProofRequest:

```json
{
  "hv_a2p_version": "1.0",
  "request_id": "<uuidv7>",
  "requesting_agent": "<agent identity string>",
  "timestamp": "<RFC 3339 UTC>",
  "minimum_trust_tier": "<T1 | T2 | T3>",
  "nist_pulse_index": "<current NIST Beacon pulse index>",
  "challenge": "<32 bytes random hex>"
}
```

The `nist_pulse_index` binds the request to a specific moment in time. The `challenge` is a random value the presenting agent must sign to prove live possession of its key.

---

## 5. Proof Presentation

The presenting agent responds with a ProofPresentation:

```json
{
  "hv_a2p_version": "1.0",
  "request_id": "<echo of request_id>",
  "presenting_agent": "<agent identity string>",
  "timestamp": "<RFC 3339 UTC>",
  "trust_tier": "<T1 | T2 | T3>",
  "proof": {
    "proof_record_id": "<PR-YYYY-NNNNN>",
    "proofregister_uri": "https://proofregister.com/record/<id>",
    "proofstamp_token": "<hex | null>",
    "root_hash": "<sha256:hex>",
    "signer_key": "<hex Ed25519 public key>"
  },
  "challenge_response": "<Ed25519 signature of challenge hex>"
}
```

The `challenge_response` proves the presenting agent controls the key that signed its receipts. A valid challenge response plus a valid ProofRegister™ record constitutes T2. A valid ProofStamp™ token constitutes T3.

---

## 6. Verification Procedure

Upon receiving a ProofPresentation the requesting agent:

1. Verifies `challenge_response` against `proof.signer_key` and the original `challenge`
2. If T2 or T3: queries ProofRegister™ at `proof.proofregister_uri` and confirms the record exists, is not revoked, and `root_hash` matches
3. If T3: verifies `proofstamp_token` against the HACKERverse published public key at proofstamp.io
4. Records the verification result in its own ProofWitness™ witness log

If verification fails at the required tier the requesting agent must not proceed with the interaction.

---

## 7. ProofRegister™ Integration

ProofRegister™ (proofregister.com) is the canonical public ledger that T2 and T3 verification depends on. It exposes a simple query API:

```
GET https://proofregister.com/record/<proof_record_id>
```

Returns the ProofBundle™ metadata, root hash, anchor timestamp, and revocation status. No authentication required for queries.

ProofRegister™ is not required for receipt validity. Receipts are independently verifiable offline. ProofRegister™ is required for T2 and T3 agent trust verification because it provides the permanent independent record that neither agent controls.

---

## 8. ProofWitness™ Witness Layer

ProofWitness™ is the shadow attestation layer that witnesses agent runtime behavior and assembles ProofBundles. In the context of PP-A2P™:

- ProofWitness™ observes agent interactions in real time
- For each interaction it assembles a ProofBundle™ containing receipt, pubkey, and verifier output
- The ProofBundle™ is anchored to ProofRegister™
- The resulting Proof Record ID is available for use in future ProofPresentations

ProofWitness™ enables continuous behavioral attestation rather than point-in-time certification. An agent with ProofWitness™ deployed can present fresh proof of its most recent behavior rather than a stale benchmark result.

This is the architectural difference between product certification and agent attestation. Product certification is periodic. Agent attestation is continuous.

---

## 9. Conformance

An implementation is conformant with PP-A2P™ if:

- ProofRequest includes all required fields including `nist_pulse_index` and `challenge`
- ProofPresentation includes a valid `challenge_response` signed with the agent's key
- T2 verification queries ProofRegister™ and confirms record existence and root hash match
- T3 verification confirms ProofStamp™ token against HACKERverse public key
- Failed verification at the required tier results in the requesting agent declining to proceed

---

## 10. References

- PP-SPEC-001 Proof Protocol™ Specification: https://github.com/proofprotocol/Defensible-Knowledge-Proof
- PP-SPEC-003 ProofBundle™ Format: https://github.com/proofprotocol/proofbundle-spec
- PP-SPEC-006 Proof of Efficacy Score: https://github.com/proofprotocol/pes-spec
- NIST Randomness Beacon: https://beacon.nist.gov
- ProofRegister™: https://proofregister.com
- ProofStamp™: https://proofstamp.io

---

## 11. Authors

Craig Ellrod, Founder & CEO, Nebulonium, Inc. (d/b/a HACKERverse)  
Castle Rock, Colorado  
2026-07-13

---

*CC BY 4.0 - Attribution to Craig Ellrod / Nebulonium, Inc. required.*
