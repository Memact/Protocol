# Epic: Federation & Cross-Provider Synchronization
## Repository: Protocol
## Epic Mapping Document

This document outlines the detailed issues to be created in the issue trackers of the four repositories of the Memact family: `Protocol`, `Contracts`, `Access` (Identity Provider), and `SDK`.

---

## 1. Protocol Repository (`Protocol`)

### Issue PROTOCOL-301: Draft RFC-006: Federated State Synchronization Protocol
* **Type**: Feature / Specification
* **Priority**: High
* **Status**: Completed
* **Description**:
  Create the formal protocol specification ([RFC-006](RFC-006-Federated-State-Sync.md)) detailing the cross-provider context state syncing architecture, including the peer handshake protocol, incremental delta synchronizations (push/pull models), validation workflows, and conflict resolution rules.
* **Acceptance Criteria**:
  * [RFC-006](RFC-006-Federated-State-Sync.md) markdown document is merged in `specs/`.
  * Fully specifies the sequence of message exchanges for syncing (handshake, push/pull deltas, single-leaf claims).
  * Specifies conflict resolution rules integrating provider weights and validator-signed logical timestamps.

### Issue PROTOCOL-302: Define JSON Schemas for Federated Syncing
* **Type**: Feature / Schema Definition
* **Priority**: High
* **Status**: Completed
* **Description**:
  Create JSON schemas for [`zk-sync-request.schema.json`](../schemas/zk-sync-request.schema.json) and [`zk-sync-response.schema.json`](../schemas/zk-sync-response.schema.json) to enable automated structural validation of sync protocol messages.
* **Acceptance Criteria**:
  * JSON schema files created in `schemas/`.
  * Validate that handshake, delta sync requests/responses, and claim verification proof structures conform to schemas.

### Issue PROTOCOL-303: Abstract Cryptographic Verification for Federation Sync (Issue #32)
* **Type**: Feature / Refactor
* **Priority**: High
* **Description**:
  Abstract cryptographic proof verification models across the Federation & Cross-Provider Synchronization protocol to be proving-system agnostic, replacing hardcoded Groth16 (`pi_a`, `pi_b`, `pi_c`) structures with generalized `proving_scheme` and `proof_data` fields.
* **Acceptance Criteria**:
  * [RFC-006](RFC-006-Federated-State-Sync.md) details scheme-agnostic batch proof verification (Groth16, PLONK, Halo2).
  * [`zk-sync-request.schema.json`](../schemas/zk-sync-request.schema.json) and [`zk-sync-response.schema.json`](../schemas/zk-sync-response.schema.json) schemas use generalized proof payloads (`proving_scheme`, `proof_data`).

---

## 2. Contracts Repository (`Contracts`)

### Issue CONTRACTS-401: Implement Cross-Provider State Delta Verification
* **Type**: Feature
* **Priority**: High
* **Status**: In Progress
* **Description**:
  Implement helper functions or wrappers to verify a chain of consensus block state transitions. The library must accept a starting state root, a sequence of blocks with corresponding batch proofs and signatures, and verify that the progression to the latest state root is cryptographically valid.
* **Acceptance Criteria**:
  * Provide verification interfaces for syncing providers to validate catch-up sequences.
  * Include tests verifying valid block progressions and catching invalid proof injections.

### Issue CONTRACTS-402: Implement Cross-Provider Dispute Resolution Circuit
* **Type**: Feature
* **Priority**: High
* **Status**: Draft
* **Description**:
  Implement a ZK circuit that verifies the correct execution of cross-provider conflict resolution. The circuit should take conflicting state leaves, their respective contributor authority trust weights, and validator-signed timestamps as private inputs, and prove that the resulting leaf selection matches the defined resolution policies.
* **Acceptance Criteria**:
  * Generate verification key for cross-provider dispute resolution circuit.
  * Proves selection without exposing raw claim values.

---

## 3. Access Repository (Reference IdP / Validator Node: `Access`)

### Issue ACCESS-501: Implement Cross-Provider Sync Handshake Endpoint
* **Type**: Feature
* **Priority**: High
* **Status**: In Progress
* **Description**:
  Add a mutual authentication endpoint `/sync/handshake` that receives peer credentials, validates identity/authorization tokens (e.g., using mTLS or signed JWTs), negotiates synchronization parameters, and verifies the user's federation consent token.
* **Acceptance Criteria**:
  * Establish secure sync session between trusted providers.
  * Correctly reject handshakes with invalid signatures or unauthorized federation requests.

### Issue ACCESS-502: Implement Delta Synchronization Sync Engine
* **Type**: Feature
* **Priority**: Critical (Blocker for sync flow)
* **Status**: In Progress
* **Description**:
  Build the push/pull delta sync engine. Pull endpoints must expose `/sync/deltas` returning consensus blocks since a specified height. Push clients must push newly minted blocks/CTTs to registered federated peers.
* **Acceptance Criteria**:
  * Local database state can reconcile transitions incrementally.
  * Successfully query and integrate blocks from peer provider node.

### Issue ACCESS-503: Implement Cross-Provider Conflict Resolution & Dispute Manager
* **Type**: Feature
* **Priority**: High
* **Status**: Draft
* **Description**:
  Build the dispute resolver that resolves concurrent updates on the same claim key from different providers. If a collision is detected, apply the authority weight and timestamp checks. Generate the dispute proof using the dispute resolution circuit.
* **Acceptance Criteria**:
  * Canonicalize correct state roots using resolution rules.
  * Disseminate dispute resolution proof to peers to settle state divergence.

### Issue ACCESS-504: Implement User Consent Token (UCT) Authentication Middleware
* **Type**: Security / Middleware
* **Priority**: High
* **Status**: Draft
* **Description**:
  Implement security middleware in the Access provider node to parse, cryptographically verify, and enforce User Consent Tokens (UCTs) on all incoming `/sync/*` requests.
* **Acceptance Criteria**:
  * Validate JWS signature against user's public key registered in the local SMT.
  * Enforce requested synchronization scopes, target provider client IDs, and token expiration.

---

## 4. Client SDK Repository (`SDK`)

### Issue SDK-601: Support Cross-Provider User Federation Consent
* **Type**: Feature
* **Priority**: Medium
* **Status**: Draft
* **Description**:
  Extend the client SDK to generate and sign cryptographic user federation consent tokens (UCTs). These tokens authorize Provider A to share/sync context states with Provider B.
* **Acceptance Criteria**:
  * SDK exposes a method to create signed federation authorizations containing target provider IDs, allowed claim scopes, and epoch expiration.

### Issue SDK-602: Direct Multi-Provider Context Query Client
* **Type**: Feature
* **Priority**: Medium
* **Status**: Draft
* **Description**:
  Build client query routing that can fetch state claims from multiple federated providers and verify the SMT membership proofs against local or cached provider context state roots (CSR).
* **Acceptance Criteria**:
  * Query and combine claims from multiple providers.
  * Validate SMT membership proofs from different providers offline.

### Issue SDK-603: Local SMT Membership Proof Cache & Offline Verification Engine
* **Type**: Feature
* **Priority**: Medium
* **Status**: Draft
* **Description**:
  Implement a local client-side cache and verifier for Sparse Merkle Tree (SMT) inclusion/exclusion proofs and verified Context State Roots (CSR).
* **Acceptance Criteria**:
  * Enable SDK applications to verify context claim proofs offline against cached state roots without hitting validator endpoints repeatedly.
