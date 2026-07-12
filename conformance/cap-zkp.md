# Conformance Checklist: ZKP-Enabled CAP Provider

This checklist defines the requirements that an Identity Provider (IdP) must satisfy to be certified as a **ZKP-Enabled Context Access Protocol (CAP) Provider**.

---

## 1. Request Handling & Parsing

- [ ] **Custom Request Parameter**: The provider's `/authorize` endpoint MUST support the parsing of the `zkp_request` URL parameter when `scope` contains `cap:zkp`.
- [ ] **Schema Validation**: The provider MUST validate incoming `zkp_request` values against the `zkp-request.schema.json` schema. If validation fails, the provider MUST return an `invalid_request` error to the client redirect URI.
- [ ] **Unsupported Circuit Fallback**: If the client requests a `circuit_id` that is not supported by the provider, the authorization flow MUST terminate with an `unsupported_response_type` or custom error parameter `unsupported_zkp_circuit`.

---

## 2. User Consent & Interface

- [ ] **Privacy-focused UX**: The user consent screen MUST clearly describe the assertion being proved (e.g. "Age is older than 18") rather than showing raw attribute details.
- [ ] **Explicit Notice**: The interface MUST explicitly notify the user that their raw personal attribute value is not being disclosed to the requesting client app.

---

## 3. Prover & Proof Generation

- [ ] **Curve and Proving System**: The provider MUST use the **BN254 pairing-friendly elliptic curve** with the **Groth16** proving system.
- [ ] **Replay Protection**: The client's unique `nonce` MUST be included as a public input to the ZK circuit. The proof generated MUST bind this nonce.
- [ ] **Attribute Salting**: To prevent brute-force dictionary attacks against the public inputs, the provider MUST hash the user attribute using a high-entropy, user-specific `attribute_salt`. The public input parameter `attribute_hash` MUST be generated as:
  $$\text{attribute\_hash} = \text{Hash}(\text{attribute\_salt} \mathbin{\Vert} \text{claim\_name})$$

---

## 4. Token Issuance & Response

- [ ] **Token Payload structure**: The provider MUST structure the `cap_zkp_assertion` claim in the returned context tokens to strictly match `zkp-response.schema.json`.
- [ ] **Dynamic Key Resolution**: The response field `verification_key_uri` MUST point to a valid, publicly accessible JSON document containing the Groth16 verification key for the corresponding circuit.
- [ ] **Header Configuration**: The server hosting the verification key MUST return correct CORS headers (`Access-Control-Allow-Origin: *`) allowing client-side SDKs to retrieve verification keys directly.

---

## 5. Security & Verification Conformance

- [ ] **Assertion Verification**: The provider's reference implementations (e.g. Access) MUST pass the verification tests defined in the `Contracts` library.
- [ ] **Negative Proving**: The provider MUST fail proof generation or return an error if the user's private attribute does not satisfy the logic constraint (e.g., if a user is under 18 and requesting an over-18 range proof).
