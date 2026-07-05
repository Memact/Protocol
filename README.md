# Memact Protocol

Protocol contains the specifications and RFC drafts for the open identity context protocols.

## What Protocol Does

Protocol defines how applications and identity providers communicate.

It covers three protocol families:

| Protocol | Purpose |
|----------|---------|
| **CAP** (Context Access Protocol) | How applications request user-approved context from an identity provider. |
| **CCP** (Context Contribution Protocol) | How applications contribute new observations and evidence to an identity provider. |
| **CRP** (Context Rectification Protocol) | How users and providers resolve conflicting or outdated context. |

The protocols are provider-independent. Any compliant identity provider can implement them.

---

## Repository Contents

- `specs/` — Formal protocol specifications and RFCs.
- `schemas/` — Machine-readable schema definitions for protocol messages.
- `conformance/` — Conformance checklists for CAP, CCP, and CRP providers.

---

## Relationship to Other Repositories

Protocol defines the interface. The other repositories implement it.

| Repository | Role |
|------------|------|
| **[Access](https://github.com/Memact/Access)** | Reference identity provider implementing CAP and CCP. |
| **[Contracts](https://github.com/Memact/Contracts)** | Shared validation schemas for protocol messages. |
| **[SDK](https://github.com/Memact/SDK)** | Client library for apps integrating with CAP and CCP. |

---

## Contributing

Contributions to the protocol specifications are welcome. Read the RFC proposal template before opening a new specification issue.

Protocol changes that affect the wire format require a formal RFC and review period before merging.

---

## License

Licensed under the Apache License 2.0.