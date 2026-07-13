# RFC 006: Federated State Synchronization Protocol

## Status
Draft (RFC-006)

## 1. Introduction & Objectives
In the Memact Protocol framework, users federate their identities and context attributes across multiple independent Identity Providers (IdPs) or sync nodes. Under the **Context Access Protocol (CAP)** and **Context Contribution Protocol (CCP)**, applications contribute claims to one provider, while relying parties query claims from another. 

To maintain consistency without compromising security, privacy, or performance, different providers must synchronize their underlying Sparse Merkle Trees (SMTs) and Context State Roots (CSRs).

This RFC proposes the **Federated State Synchronization Protocol**. It defines the handshake, data replication, state verification, and conflict resolution mechanisms required to securely sync user contexts across semi-trusted or untrusted providers in a distributed topology.

### Objectives
* **Consistency & Eventual Convergence**: Ensure federated providers converge on a single canonical Context State Root (CSR) representing the user's latest verified context.
* **Cryptographic Integrity**: Prevent any provider from injecting unauthorized or invalid state transitions during sync by enforcing Zero-Knowledge Proof validation.
* **Privacy Preservation**: Sync state updates as Sparse Merkle Tree leaf hashes, exposing raw claim values only when explicitly authorized by user-signed consent.
* **Robust Conflict Resolution**: Mathematically resolve concurrent updates from different providers using a standardized trust-weight and timestamp hierarchy.

---

## 2. Federated Trust & Handshake Protocol

Before synchronization begins, providers must establish a secure sync relationship.

```
+------------------+                        +------------------+
|    Provider A    |                        |    Provider B    |
+--------+---------+                        +--------+---------+
         |                                           |
         | -------- 1. Mutual Authentication -------> |
         | <------- 2. Connection Confirmed -------- |
         |                                           |
         | -------- 3. Send zk-sync-request --------> | (Contains User Consent Token)
         |                                           |
         |                                           | [Validate User Consent,
         |                                           |  Check State Heights]
         |                                           |
         | <------- 4. Return zk-sync-response ----- | (Negotiated Sync & Deltas)
         v                                           v
```

### 2.1. Mutual Authentication
Providers authenticate each other using Mutual TLS (mTLS) or OAuth 2.0 Client Credentials with JWTs signed by their respective node authority keys.

### 2.2. User Consent Verification
A provider cannot synchronize a user's context without a valid **User Consent Token (UCT)**. The UCT is a JSON Web Signature (JWS) signed by the user's private key containing:
* `user_identity`: The user's decentralized identifier (DID) or URI.
* `authorized_providers`: Array of provider client IDs permitted to sync this context.
* `scopes`: Allowed claim keys/namespaces to sync (e.g. `["user.location", "user.kyc"]`).
* `expiration`: Epoch expiration timestamp.

The handshake MUST include this UCT. The receiving provider verifies the signature against the user's registered public key stored in the access control leaves of the local SMT.

---

## 3. Synchronization Flow & Models

The synchronization protocol supports two main models: **Incremental State Sync (Block Rollup)** and **Direct Claim Query (Single-Leaf)**.

### 3.1. Incremental State Sync (Pull-based)
Used to catch up a lagging provider to the current ledger height.

1. **Request**: Provider A sends a `zk-sync-request` containing `request_type: "PULL_DELTAS"`, specifying its latest `since_block_number` and the hash of its `known_state_root`.
2. **Verify & Diff**: Provider B validates the request, computes the block height difference, and collects the list of `ZKConsensusBlock`s generated since that height.
3. **Response**: Provider B returns a `zk-sync-response` containing a sequence of block payloads, each containing:
   * Block number and timestamp.
   * State transitions and their associated `batch_proof`.
   * Consensus validator signatures.
4. **Transition Verification**: Provider A verifies the batch proof and signatures for each block sequentially. Only after validation does Provider A update its local SMT and CSR.

### 3.2. Real-time Synchronization (Push-based)
Used for low-latency synchronization of new context contributions.

1. When Provider A finalizes a new consensus block locally, it serializes the block structure.
2. Provider A sends a `zk-sync-request` of type `PUSH_DELTAS` containing the block to all connected federated providers.
3. Peer providers verify the batch proof, apply the state transitions to their local SMTs, and reply with a success status.

### 3.3. Direct Claim Query (Single-Leaf Pull)
Used when a provider needs to verify a single claim value on-demand without performing a full ledger catch-up.

1. **Request**: Provider A requests a specific `claim_key` for a user context.
2. **Response**: Provider B retrieves the leaf from its SMT and returns:
   * `claim_key` & `leaf_hash`.
   * SMT membership proof (sibling hashes path up to the current CSR).
   * Raw `claim_value` and `salt` (if authorized by the UCT).
3. **Verification**: Provider A validates the membership proof against its verified local copies of Provider B's state root.

---

## 4. Conflict Resolution Policy (CRP)

When concurrent contributions are submitted to different providers for the same `claim_key`, the state roots will diverge, creating a "split-brain" state.

```
                  [ Divergent State Roots ]
                  /                       \
        Provider A Root             Provider B Root
        Update: Loc = "US"          Update: Loc = "DE"
        Weight = 80                 Weight = 90
        Time = T1                   Time = T2
                  \                       /
                   \                     /
                  +-----------------------+
                  |  Dispute Resolution   |
                  |  - Compare weights    |
                  |  - Compare timestamps |
                  +-----------+-----------+
                              |
                              v (Winner Selected)
                     Provider B Update
                     Loc = "DE" (Weight 90)
```

### 4.1. Conflict Resolution Evaluation Hierarchy
To resolve collisions, sync engine nodes evaluate the following rules sequentially:

1. **Authority Trust Weight**: Each claim update is signed by a contributor identity. The SMT stores a registration mapping of contributor identities to **Trust Weights** (integer 1-100). The update from the contributor with the higher trust weight is selected as canonical.
2. **Logical Timestamp**: If trust weights are identical, the sync engine evaluates the cryptographic timestamps. Every state transition has a timestamp signed by the proposing validator node. The update with the later verified timestamp wins.
3. **Hash Determinism**: In the highly unlikely event that both trust weights and timestamps are identical, the update with the lexicographically smaller transaction signature hash wins.

### 4.2. Dispute Resolution Proofs
To avoid revealing raw values during conflict settlements, the dispute resolution is executed inside a specialized ZK circuit:
* **Private Inputs**:
  * Conflicting leaf values: $V_A$, $V_B$.
  * Associated salts: $S_A$, $S_B$.
  * Trust weights of respective contributors: $W_A$, $W_B$.
  * Cryptographic validator timestamps: $T_A$, $T_B$.
* **Public Inputs**:
  * State roots: $CSR_A$, $CSR_B$.
  * Conflict claim key: $K$.
  * Winning leaf hash: $H_{winner}$.
* **Proof Statement**: The circuit proves that $H_{winner}$ is the hash of the value that satisfies the conflict resolution policy constraints given the authority weights and timestamps of the private inputs.

Once the proof is generated, the winning provider packs it into a dispute block, and broadcasts it to converge the network.

---

## 5. Security & Privacy Considerations

### 5.1. Sybil and Replay Injections
Attackers might intercept old sync packages and replay them to revert a user's context state.
* **Mitigation**: Every sync message contains a unique, high-entropy UUID (`request_id`) and an epoch timestamp. Sync engines maintain a sliding-window cache of verified `request_id`s and reject duplicate packets.

### 5.2. Double-Syncing Prevention
If Provider A syncs with Provider B, and Provider B syncs with Provider C, infinite sync loops must be prevented.
* **Mitigation**: Sync blocks maintain an audit path list of providers that have already verified and committed the block. Providers reject sync updates containing their own identity in the audit path list.
