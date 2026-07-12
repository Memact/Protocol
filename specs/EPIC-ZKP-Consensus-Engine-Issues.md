# Epic: ZKP Consensus Engine (Zero Knowledge Proofs)
## Repository: Protocol
## Epic Mapping Document

This document outlines the detailed issues to be created in the issue trackers of the four repositories of the Memact family: `Protocol`, `Contracts`, `Access` (Identity Provider), and `SDK`.

---

## 1. Protocol Repository (`Protocol`)

### Issue PROTOCOL-201: Draft RFC-005: ZKP-Enabled Context Consensus Engine
* **Type**: Feature / Specification
* **Priority**: High
* **Description**:
  Create the formal protocol specification (RFC-005) detailing the Sparse Merkle Tree (SMT) architecture, Context Transition Transaction (CTT) structures, ZK-Rollup batch proving pipelines, and Conflict Resolution rules (CRP).
* **Acceptance Criteria**:
  * RFC-005 markdown document is merged in `specs/`.
  * Fully specifies SMT leaf constructions using Poseidon hashing.
  * Specifies sequential transitions and signature-based authority controls inside circuits.

### Issue PROTOCOL-202: Define JSON Schemas for Transaction and Consensus Block
* **Type**: Feature / Schema Definition
* **Priority**: High
* **Description**:
  Create JSON schemas for `zk-context-transition.schema.json` and `zk-consensus-block.schema.json` to enable automated structural validation.
* **Acceptance Criteria**:
  * JSON schema files created in `schemas/`.
  * Validate that root properties like `block_number`, `proof`, `public_inputs` (with proper pattern matches for 32-byte hex hashes), and signature fields match expectations.

---

## 2. Contracts Repository (`Contracts`)

### Issue CONTRACTS-301: Implement Sparse Merkle Tree (SMT) Poseidon Circuits
* **Type**: Feature
* **Priority**: Critical (Blocker for proving)
* **Description**:
  Write the core circuit logic (e.g. in Circom or Halo2) validating Sparse Merkle Tree membership. The circuit should verify Poseidon hashing over a path of depth 256.
* **Acceptance Criteria**:
  * Provide compilation artifacts for SMT inclusion verification.
  * Include tests verifying path generation and checking incorrect path rejection.

### Issue CONTRACTS-302: Implement Batch Rollup Prover Circuit
* **Type**: Feature
* **Priority**: High
* **Description**:
  Write a rollup/batch circuit that takes $N$ state transition witness elements, verifies authority signatures, checks trust weights (CRP conflict resolution rules), and outputs a combined state root update proof.
* **Acceptance Criteria**:
  * Proves transition logic mathematically inside a unified Groth16/PLONK circuit.
  * Correctly rejects updates signed by unauthorized accounts.

---

## 3. Access Repository (Reference IdP / Validator Node: `Access`)

### Issue ACCESS-401: Implement Transaction Mempool
* **Type**: Feature
* **Priority**: High
* **Description**:
  Add an incoming transaction mempool endpoint `/contribute/zkp` that receives `zk-context-transition` payloads, validates their individual structural properties, and queues them for reconciliation.
* **Acceptance Criteria**:
  * CTTs are successfully stored in a temporary buffer.
  * Validate sender signature and basic schema conformance prior to queuing.

### Issue ACCESS-402: Implement Batch Prover and Dispute Resolution Manager
* **Type**: Feature
* **Priority**: High
* **Description**:
  Build the consensus batch compiler. It must aggregate queued CTTs, run conflict resolution heuristics based on registered weight limits, generate the batch witness inputs, and run the rollup prover to produce the block's `batch_proof`.
* **Acceptance Criteria**:
  * Periodic block execution engine successfully executes.
  * Valid consensus block payloads are generated according to `zk-consensus-block.schema.json`.

### Issue ACCESS-403: Implement Validator Consensus Signer
* **Type**: Feature
* **Priority**: Medium
* **Description**:
  Implement block verification and signature collection. Validators must verify incoming block proofs off-chain and sign the block header upon successful validation.
* **Acceptance Criteria**:
  * Nodes broadcast signatures when proof validates.
  * State root is canonicalized once signatures meet quorum threshold.

---

## 4. Client SDK Repository (`SDK`)

### Issue SDK-501: Implement SMT Local Leaf Hash Builder
* **Type**: Feature
* **Priority**: High
* **Description**:
  Add local client utilities to construct and salt claim leaf values matching Poseidon hash criteria.
* **Acceptance Criteria**:
  * Expose function `buildLeafHash(key, value, salt)` returning corresponding Poseidon output.

### Issue SDK-502: CTT Submission Client
* **Type**: Feature
* **Priority**: Medium
* **Description**:
  Build client transaction wrapper and signer. Enables authorized SDK instances to sign updates, generate inclusion paths, and post CTT objects directly to validator mempools.
* **Acceptance Criteria**:
  * SDK exposes `.submitContextContribution(claimKey, newValue, provingWitness)`.
