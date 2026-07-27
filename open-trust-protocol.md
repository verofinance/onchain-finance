# Open Trust Protocol

**Portable organization identity with verifiable credentials, consented disclosure, and encrypted evidence anchored to public blockchain networks.**

## Abstract

This document specifies the Open Trust Protocol, a shared trust layer for identity, credentials, attestations, consent, and proofs. An organization identity is an account on a public blockchain network, controlled by cryptographic keys the holder manages and resolvable to a public record of its current controller keys. Trusted issuers sign verifiable credentials about the identity; the credential content stays with the holder, encrypted, while a commitment (hash, issuer, claim type, status reference, expiry) anchors on-chain. Holders authorize disclosure through signed, scoped, expiring consent grants. Document-based compliance workflows (requirement discovery, encrypted document records, signed data extraction, submissions, and structured information requests) are first-class protocol objects rather than a legacy path. Verifiers check issuer signatures and revocation status against on-chain registries without contacting the issuer, so verification works even if the issuer is offline or ceases to exist.

The protocol relies on open standards but is not limited to any single one. Internet standards supply identifiers, credentials, and authentication; blockchain standards supply programmable accounts, attestation registries, and status. The protocol defines the properties every layer must provide and leaves the choice of concrete standard to implementation profiles, so the system stays interoperable across internet and blockchain standards.

The protocol is shared infrastructure. Implementations can optionally provide
open-source applications through which companies manage identities,
credentials, consent grants, submissions, and information requests. Such
applications can be embedded or extended by providers without changing the
protocol objects. The protocol is intended to reduce onboarding duplication,
enforce permissions, and make compliance evidence portable and auditable
without turning sensitive content into public plaintext.

## Motivation

### Onboarding and information exchange do not compose

Every financial provider runs its own verification of the same counterparty: the same incorporation documents, ownership structure, and bank statements are uploaded, reviewed, and stored again for each relationship. The result of one provider's review is not portable to the next. Businesses pay for this in weeks of onboarding per corridor, providers pay in duplicated compliance work, and the wider ecosystem is left with fragmented, inconsistent views of the same entity.

Review does not end at submission. Providers come back with requests for information: clarifications, additional documents, updated evidence. Today those exchanges live in email threads and ticket systems, disconnected from the identity they concern. An answer given to one provider is not reusable for the next, evidence loses its version history, and neither side can later prove what was asked, what was delivered, and when. Managing this information exchange is as much a part of the trust problem as the initial verification, and no shared infrastructure exists for either.

### Authentication is solved; portable identity is not

Modern authentication standards such as [WebAuthn](https://www.w3.org/TR/webauthn-3/) (passkeys) removed shared secrets: a device holds a private key, and the user proves control with a signature. Blockchain networks rest on the same primitive with different curves (secp256k1 on Ethereum, ed25519 on Stellar, secp256r1 in WebAuthn authenticators): an account is a key pair, and control is proven by signature. But each WebAuthn key is bound to the domain that registered it, so the identity it protects belongs to the relying party, not to the holder; and a bare blockchain key controls assets, not a verified identity. Every service still runs its own registration and accumulates its own isolated view of who the user is.

The protocol generalizes this model rather than replacing it: holder-controlled keys bind to an account on a public network instead of a domain. One identity, and every credential, attestation, and consent grant attached to it, becomes verifiable by any application, with revocation and status published on registries no single party operates.

### Compliance runs on documents today

No issuer network exists yet that could replace document review with pure credential exchange, and regulated providers will keep requiring source documents for years regardless. Providers request original files, extract structured data from them, review the result under their own policy, and come back with follow-up questions. A trust protocol that only speaks verifiable credentials cannot onboard anyone today. The protocol therefore specifies document workflows as normative primitives alongside credentials, and defines a provenance ladder from self-declared data to proof-only disclosure so that each verifier interaction upgrades data quality without requiring the whole market to move at once.

## Specification

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHOULD”, “SHOULD
NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be
interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119)
and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174).

### Reliance on open standards

The protocol composes open standards rather than defining new primitives, and it does not bind to any single one. Normative requirements in this specification are expressed as properties (holder-controlled keys, offline-verifiable credentials, issuer-independent status) and name concrete standards (W3C DID Core, Verifiable Credentials 2.0, WebAuthn, ERC-4337, CAP-51) only as recommended or candidate implementations of those properties. A deployment selects a profile of concrete standards; two implementations interoperate when they share a profile. New standards MAY be adopted into profiles without changing the protocol objects. The intent is an open system that interoperates across internet and blockchain standards without committing to either alone.

### Actors

| Actor              | Role                                                                                                                       |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------- |
| Identity holder    | Company, individual officer, beneficial owner, service provider, issuer, vault LP, or borrower.                            |
| Credential issuer  | KYB provider, auditor, financial institution, logistics provider, custodian, registry, or licensed originator.             |
| Verifier           | Application, lender, vault manager, marketplace, counterparty, or regulator with consented access.                         |
| Trusted attestor   | Approved party that signs facts about status, documents, events, collateral, reserves, or performance.                     |
| Processing service | Consented service that extracts, validates, or translates document data and signs the provenance of the result.            |
| Policy authority   | Governance or compliance function that defines eligible issuers, claim types, revocation rules, and jurisdiction controls. |

### Protocol objects

| Object                    | Description                                                                                          | On-chain footprint                                                              |
| ------------------------- | ---------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| `TrustIdentity`           | Organization or person identity controlled by authorized keys.                                       | DID, wallet binding, controller references, key rotation events.                |
| `CredentialClaim`         | A verified assertion about an identity.                                                              | Credential hash, issuer, claim type, status reference, expiry.                  |
| `ConsentGrant`            | Permission from a holder to a verifier for specific claims or data rooms.                            | Scope, verifier, expiry, revocation status.                                     |
| `Attestation`             | Signed statement by a trusted party about an event or state.                                         | Attestor, subject, claim, evidence commitment, timestamp.                       |
| `EncryptedEvidenceRecord` | Encrypted evidence such as invoices, contracts, bills of lading, audit reports, or compliance files. | Ciphertext reference, hash, Merkle root, access policy, version reference.      |
| `ProofRequest`            | Request for a holder to prove eligibility or a threshold claim.                                      | Claim topic, policy id, verifier, proof result reference.                       |
| `RequirementSet`          | Machine-readable list of fields, document types, and terms a verifier requires in a given context.   | Verifier, policy version, schema reference.                                     |
| `ExtractedDataRecord`     | Structured fields derived from documents by a consented processing service.                          | Data hash, source document references, extractor signature, provenance level.   |
| `Submission`              | Package of claims, data, and documents delivered to one verifier under a consent grant.              | Verifier, scope reference, status, decision reference.                          |
| `InformationRequest`      | Structured follow-up case a verifier opens during review.                                            | Verifier, subject reference, status, message commitments, resolution timestamp. |

### Layered architecture

The protocol organizes into five layers. Each layer depends only on the layers below it.

```text
┌─────────────────────────────────────────────────────────┐
│  Applications: onboarding, lending, payments, markets   │
├─────────────────────────────────────────────────────────┤
│  Verification: policy checks, proof requests, audit     │
├─────────────────────────────────────────────────────────┤
│  Consent: scoped grants, expiry, revocation             │
├─────────────────────────────────────────────────────────┤
│  Credentials & evidence: VCs, attestations, encrypted   │
│  records, status registries                             │
├─────────────────────────────────────────────────────────┤
│  Identity: smart accounts, DIDs, controller keys        │
└─────────────────────────────────────────────────────────┘
```

### Identity

A `TrustIdentity` is a smart account on a supported network.

- Controller keys are cryptographic keys held by the organization's officers. The account MUST accept the signer types its network profile supports; device-held signers such as WebAuthn passkeys are RECOMMENDED because they remove seed phrases from the officer experience.
- The account MUST enforce the organization's signing policy: single officer, quorum, or role-based scopes. Identity-level changes (key rotation, policy changes) SHOULD require a higher approval threshold than routine consent grants.
- The identity MUST resolve, from the identifier alone, to a public record listing current controller keys, verification methods, and service endpoints, so any verifier can discover current keys and their authority. A registry-backed DID document conforming to [DID Core](https://www.w3.org/TR/did-core/) and supporting those properties is the RECOMMENDED representation. [`did:pkh`](https://github.com/w3c-ccg/did-pkh/blob/main/did-pkh-method-draft.md) MAY be used as an immutable blockchain-account alias only alongside a separate mutable resolution mechanism, because `did:pkh` does not itself provide service endpoints or key rotation.
- Key rotation MUST NOT change the account address. Credentials, attestations, and history attach to the address and survive controller changes.
- Recovery MUST NOT hand unilateral control to any single third party. Multi-party recovery following the [SEP-30](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0030.md) model is RECOMMENDED.

Two initial network profiles illustrate the pattern. On Ethereum networks,
identities can be [ERC-4337](https://eips.ethereum.org/EIPS/eip-4337) smart
accounts with secp256r1 signature validation through the
[EIP-7951](https://eips.ethereum.org/EIPS/eip-7951) `P256VERIFY` precompile and
[EIP-1271](https://eips.ethereum.org/EIPS/eip-1271) contract-signature support,
where the selected network exposes those capabilities. On Stellar, identities
can be Soroban custom account contracts using secp256r1 verification introduced
in [CAP-51](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0051.md).
Session authentication with off-chain applications can use
[EIP-4361 (Sign-In with Ethereum)](https://eips.ethereum.org/EIPS/eip-4361) and
[SEP-10](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0010.md),
respectively. Other profiles MAY be defined without changing the protocol
objects.

### Credentials and status

Issuers sign structured claims about an identity. The credential format MUST be an open standard verifiable without contacting the issuer. The [W3C Verifiable Credentials Data Model v2.0](https://www.w3.org/TR/vc-data-model-2.0/), secured with [VC Data Integrity](https://www.w3.org/TR/vc-data-integrity/) or [JOSE/COSE](https://www.w3.org/TR/vc-jose-cose/) proofs (including SD-JWT for selective disclosure), is the RECOMMENDED format; on-chain attestation formats MAY serve the same role for claims that are public by design.

- Credential content MUST stay with the holder, encrypted. Only a commitment (credential hash, issuer, claim type, status reference, expiry) is anchored on-chain.
- Every credential MUST carry a status reference. Status MUST be checkable without contacting the issuer; [Bitstring Status List v1.0](https://www.w3.org/TR/vc-bitstring-status-list/) with on-chain commitments is RECOMMENDED, so a verifier checks revocation against a compressed public list without revealing to the issuer which credential it is checking.
- Verifiers MUST check issuer authority against the issuer registry maintained by the policy authority, and MUST check status at time of use. A verifier that cannot resolve status MUST fail closed.
- Issuers MUST be able to suspend or revoke a credential with a single status update, visible to every relying application without per-application notification.

Claim topics include: identity and control (legal name, jurisdiction, registration number, authorized signers, beneficial ownership); compliance status (KYB approval, sanctions screening, adverse media, PEP checks); financial status (audited financials available, bank account validation, tax status, payment-rail eligibility); operational status (logistics, custody, insurance, shipment and goods-control events); investor status (accredited, qualified, jurisdiction-eligible, whitelisted wallet); and protocol status (issuer approval, attestor approval, vault eligibility, revocation status).

### Consent

Disclosure requires an explicit `ConsentGrant`.

- A grant MUST be signed by the holder's account under its signing policy, and MUST specify verifier, claims or data rooms in scope, purpose, and expiry.
- Grants MUST be recorded so both parties can later prove, to a regulator or a court, exactly what was shared, when, and under whose authority.
- Revocation is a signed state change and MUST immediately prevent new retrievals
  or authorizations under the grant. Verifiers MUST stop future access when a
  grant is revoked or expires. Revocation cannot recall copies already
  disclosed; previously received data remains subject to the grant's purpose
  limits and to applicable retention, deletion, audit, and legal obligations.
- Grants MUST use a typed, human-readable, replay-safe signing format; [EIP-712](https://eips.ethereum.org/EIPS/eip-712) typed structured data is the RECOMMENDED format on Ethereum networks.

The protocol defines three disclosure levels:

| Level                 | Pattern                                           | Example                                                     |
| --------------------- | ------------------------------------------------- | ----------------------------------------------------------- |
| Public state          | Status is visible to anyone.                      | Credential exists and is not revoked.                       |
| Selective disclosure  | Verifier sees only chosen claims.                 | Company is incorporated in Brazil and KYB-approved.         |
| Proof-only disclosure | Verifier gets a proof that a policy is satisfied. | Revenue exceeds a threshold without revealing full revenue. |

A holder MUST be able to authorize a lender to verify "KYB approved and sanctions clear" without exposing shareholders, bank statements, or contracts.

### Evidence and document workflows

Document-based flows are normative protocol behavior, not an extension.

- **Requirement discovery.** A verifier publishes a machine-readable `RequirementSet`: which fields, document types, and terms it requires in a given context. Requirements differ per verifier and per context; the protocol standardizes how they are expressed, not what they contain.
- **Document records.** Originals upload encrypted into the holder's data room, versioned, with commitments anchored on-chain. Verifiers holding a consent grant receive the actual files, not only claims about them, because their compliance programs require source documents.
- **Data extraction.** Consented processing services parse documents into structured fields and MUST sign the result. Every field in a profile MUST carry provenance: self-declared, extracted from a named document by a named service, or verified by an issuer.
- **Submissions.** A `Submission` packages claims, structured data, and documents for one verifier under a scoped grant, with its own status and decision reference.
- **Information requests.** When a verifier needs clarification during review, it opens an `InformationRequest` against the submission: a threaded case with messages, attachments, status, and resolution, recorded against the identity instead of scattered across email. Responses enter the data room as versioned evidence, available for reuse.
- **Expiry and freshness.** Evidence ages: certificates lapse, statements go stale, and registrations come up for renewal. Document records, extracted fields, and credentials SHOULD carry a validity or review date, and expiry MUST be visible to relying verifiers alongside provenance. Approaching expiry raises a renewal request to the holder, so a profile stays continuously up to date instead of being re-collected at each new relationship.

These primitives define a provenance ladder:

```text
Level 0   Self-declared
Level 1   Document-backed (encrypted original, on-chain commitment)
Level 2   Extraction-signed (named processing service, signed result)
Level 3   Issuer-verified (verifiable credential)
Level 4   Proof-only (policy satisfied without disclosure)
```

Each verifier interaction pushes data up the ladder; no rung requires the market to reach the top before the protocol is useful. The primitives are application-neutral; their composition into a multi-provider onboarding flow (requirement sets per corridor, provider review, RFI resolution, decision reuse) is specified by the [Global KYB Network](./global-kyb-network.md).

### Verification

A verification exchange proceeds as follows:

1. The verifier sends a `ProofRequest` naming the claims or thresholds it requires (e.g. KYB approval, sanctions status, jurisdiction).
2. An authorized officer approves the request with a signature under the account's signing policy, producing a scoped `ConsentGrant` recorded against the identity.
3. The holder delivers the credentials, documents, or proof results the grant allows.
4. The verifier checks issuer signatures against the issuer registry, commitments against the chain, and status against the revocation list. The issuer is not contacted and does not learn where the credential is used.
5. The verifier records an access event, completing the audit trail, and makes its own compliance decision. The protocol supplies verified inputs, not approvals.
6. On a later status update (suspension, revocation), the verifier's next status check MUST fail closed.

Two properties of this flow are normative:

- **The issuer is not in the verification path.** Verification MUST keep working if the issuer's systems are offline or if the issuer ceases to exist.
- **Consent is a recorded artifact.** Every disclosure is a signed, scoped, expiring grant that both parties can later prove, rather than a checkbox ticked during onboarding.

### Lifecycle

| State                 | Entered when                             | Exits to                                                                   |
| --------------------- | ---------------------------------------- | -------------------------------------------------------------------------- |
| `DraftIdentity`       | Account created, profile incomplete.     | `PendingVerification` on profile submission.                               |
| `PendingVerification` | Profile submitted to an issuer.          | `Verified` when the issuer signs credentials; `Rejected` on failed checks. |
| `Verified`            | Credentials issued.                      | `Active` when wallet and consent are ready.                                |
| `Active`              | Identity operational.                    | `CredentialUpdated` (returns to `Active`), `Suspended`, or `Revoked`.      |
| `Suspended`           | Sanctions hit, expiry, or investigation. | `Active` when remediation is approved.                                     |
| `Revoked`             | Identity closed or invalidated.          | Terminal.                                                                  |

## Rationale

### Why compose open standards rather than invent primitives

The protocol introduces no new cryptographic scheme and no proprietary format. Internet standards already define identifiers, credentials, and authentication with the properties a trust layer needs; blockchain standards already define programmable accounts, attestation registries, and status that no single party operates. The protocol's contribution is the composition: credentials from the internet-standards world anchored to identities, registries, and settlement from the blockchain world, together with a deliberate refusal to hard-bind any layer to one standard. Requirements name properties; profiles name standards. This keeps the system interoperable across both worlds and lets implementations adopt better standards as they emerge without breaking the protocol objects.

Passkeys illustrate the approach. Where a profile uses WebAuthn signers, an
organization can authenticate with device-held credentials instead of a seed
phrase. Embedded-wallet products demonstrate that key-based accounts can sit
behind familiar login flows, but no particular wallet vendor is required by the
protocol. Passkeys are one profile choice rather than the foundation on which
every implementation depends.

### Why public blockchain networks

A protocol like this could be attempted as a federation of company-run servers. Previous attempts at federated business identity have struggled for the same structural reasons.

- **Neutrality.** A trust registry operated by one company is a competitive threat to every other participant; banks do not build on a rival's identity database. Public networks have no owner, so competitors can rely on the same registry without ceding ground to each other. This is the property that made TCP/IP and TLS adoptable.
- **Persistence.** Passkeys bind to domains, and domains are leases: the credential dies with the relying party. Federated identity dies with the identity provider. An identity anchored to a public network outlives any issuer, application, or vendor, which is the minimum standard for records that back multi-year financial obligations.
- **Verification without a phone-home.** In federated systems, verifying a claim means calling the issuer, which creates an availability dependency and leaks usage data: the issuer learns every place the business banks. On-chain commitments and status lists let any verifier check validity locally. The issuer is consulted at issuance, never at use.
- **Revocation that propagates.** In bilateral integrations, revocation is an email. On a shared network, a single status update is immediately visible to every application, and verifiers fail closed.
- **Programmable key management.** Smart accounts support quorum control, key
  rotation, session scoping, and recovery policies. CAP-51 and EIP-7951 allow
  compatible profiles to verify the P-256 signature component of a WebAuthn
  assertion; they do not replace the rest of the WebAuthn ceremony.
- **Trust and settlement share a substrate.** When credentials, consent, and assets live in the same execution environment, eligibility checks compose atomically with settlement: a permissioned transfer can verify the holder's credential status in the same transaction that moves the asset. No federation of identity servers can offer that.

### Why document workflows are normative

Credential-only designs assume an issuer network that does not exist yet, and regulated verifiers will keep requiring source documents regardless. Specifying requirement sets, encrypted document records, signed extraction, submissions, and information requests as protocol objects lets applications run today's compliance flows on the trust layer from day one, while the provenance ladder gives every interaction a path toward credential- and proof-based verification. A parsing error that propagates into reusable records is worse than a rejected document, which is why extraction results carry a mandatory signature and provenance level.

### Why Stellar and Ethereum

Stellar contributes protocol-native P-256 verification through CAP-51, an
anchor ecosystem with KYC exchange conventions such as SEP-12, and settlement
rails designed for cross-border payments. Ethereum contributes widely used
smart-account, signature, and attestation standards, including ERC-4337,
EIP-712, EIP-1271, EIP-7951, and registries such as the
[Ethereum Attestation Service](https://attest.org/). The protocol defines
profiles for both networks with shared schemas and credential formats; neither
network is required for every deployment.

### Feasibility of each requirement

| Requirement                                       | State of support                                                                                                                         |
| ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| On-chain secp256r1 verification                   | Native in Soroban since Stellar Protocol 21 (CAP-51); EIP-7951 defines Ethereum’s `P256VERIFY` precompile at a cost of 6,900 gas.        |
| Programmable accounts (quorums, rotation, scopes) | ERC-4337 smart accounts on Ethereum; Soroban custom account contracts on Stellar.                                                        |
| DID resolution to network accounts                | `did:pkh` and chain-specific DID methods; W3C DID Core.                                                                                  |
| Credential status without issuer dependency       | Bitstring Status List anchored on-chain; attestation-registry revocation patterns.                                                       |
| Issuer governance                                 | Protocol-level policy authority; requires legal agreements and removal procedures beyond the technical registry.                         |
| Encrypted storage with on-chain commitments       | Encrypted off-chain or decentralized storage; hashes, Merkle roots, and access policies on-chain.                                        |
| Organizational key recovery                       | Multi-party recovery patterns (SEP-30 model; social and institutional recovery in smart accounts).                                       |
| Legal enforceability                              | Requires standardized legal terms incorporated by reference into protocol objects; jurisdiction-dependent.                               |
| Shared claim schemas                              | VC 2.0 data model plus a protocol-governed schema registry; the main coordination task, not a technical one.                             |

Where a requirement is only partially met, the remaining gap is operational rather than cryptographic.

### Zero-knowledge proofs

Proof-only disclosure is a defined level of the protocol, and the specification is deliberately neutral about the machinery behind it. The protocol neither commits to a particular proof system nor defers the capability: profiles adopt zero-knowledge proofs where they reduce real disclosure risk, and selective disclosure (SD-JWT) satisfies the same interface where it is sufficient. Verifiers consume a proof result and a policy reference either way.

## Backward Compatibility

The protocol changes no existing standard; it composes them.

- **WebAuthn.** Authenticators, browsers, RP ID scoping, the WebAuthn ceremony, and platform passkey UX remain unchanged. A compatible profile additionally registers the credential public key or identifier with smart-account authorization logic; this supplements rather than replaces the web relying party.
- **SEP-12.** Today each Stellar anchor collects and stores KYC data independently. The protocol upgrades this flow so a SEP-12 exchange references verifiable credentials and consented data-room access instead of re-uploaded documents. Anchors that do not adopt the protocol continue to work unchanged.
- **Existing compliance programs.** KYB providers become credential issuers without abandoning document review; the document workflow primitives mirror the requirement-discovery, collection, extraction, and RFI processes providers already run. Verifiers keep full responsibility for their own compliance decisions.
- **Networks.** No consensus or protocol change is required on Ethereum or Stellar; the protocol deploys as contracts and off-chain conventions on shipped infrastructure.

## Security Considerations

- **Schema fragmentation.** Credentials fragment if each issuer names claims independently. A protocol-governed schema registry is the main coordination task; without it, verifiers silently accept semantically different claims under one name.
- **Issuer governance.** A credential is only as trustworthy as the registry that lists its issuer. Admission, monitoring, and removal of issuers need legal agreements and procedures beyond the technical registry, including a dispute process for incorrect revocations.
- **Revocation latency.** Sanctions, fraud, and authority changes must propagate in minutes. Verifiers MUST check status at time of use and fail closed when status cannot be resolved.
- **Key compromise and recovery.** Officers leave and devices are lost. Quorum policies, rotation without address change, and multi-party recovery bound the blast radius of a single compromised passkey, but corporate signer changes also need legal and operational procedures.
- **WebAuthn assertion validation.** P-256 signature verification alone is not WebAuthn authentication. An implementation that uses a WebAuthn profile MUST validate each applicable assertion-verification step in [WebAuthn Level 3 §7.2](https://www.w3.org/TR/webauthn-3/#sctn-verifying-assertion), including, without limitation, the challenge, origin, RP ID hash, credential identity, user-presence and user-verification flags, authenticator data, and client-data binding. Implementations SHOULD require canonical signatures or ensure that replay protection and artifact identifiers do not depend on signature-byte uniqueness.
- **Extraction provenance.** Extraction errors can propagate into reusable records and outlive the document that produced them. Mandatory extractor signatures and provenance levels let downstream parties see exactly what they are relying on.
- **Consent comprehensibility.** A grant a non-technical operator cannot understand is not meaningful consent. Grant scopes must be human-readable (typed structured data such as EIP-712) and enforceable by applications, not only recordable.
- **On-chain metadata.** Only commitments and status anchor on-chain, but commitment graphs can still leak relationship patterns. Status list designs that hide which credential is being checked (bitstring lists) limit this leakage.
- **Verification mechanism availability.** A profile MUST state which signature-verification mechanism it requires. If the profile-required mechanism is unavailable, an implementation MUST reject the signature unless the profile declares an available fallback. Any fallback MUST provide the same signature-validation and acceptance semantics as the primary mechanism.
- **Legal enforceability.** A consent grant or credential must map to obligations a court recognizes; standardized legal terms incorporated by reference are jurisdiction-dependent and unproven at scale.

## References

### Identity and credential standards

- [W3C Verifiable Credentials Data Model v2.0](https://www.w3.org/TR/vc-data-model-2.0/)
- [W3C Decentralized Identifiers (DIDs) v1.0](https://www.w3.org/TR/did-core/)
- [W3C Bitstring Status List v1.0](https://www.w3.org/TR/vc-bitstring-status-list/)
- [W3C Verifiable Credential Data Integrity 1.0](https://www.w3.org/TR/vc-data-integrity/)
- [W3C Securing Verifiable Credentials using JOSE and COSE](https://www.w3.org/TR/vc-jose-cose/)
- [W3C Web Authentication (WebAuthn) Level 3](https://www.w3.org/TR/webauthn-3/)

### Ethereum

- [ERC-4337: Account Abstraction Using Alt Mempool](https://eips.ethereum.org/EIPS/eip-4337)
- [EIP-7951: Precompile for secp256r1 Curve Support](https://eips.ethereum.org/EIPS/eip-7951)
- [EIP-1271: Standard Signature Validation Method for Contracts](https://eips.ethereum.org/EIPS/eip-1271)
- [EIP-712: Typed Structured Data Hashing and Signing](https://eips.ethereum.org/EIPS/eip-712)
- [EIP-4361: Sign-In with Ethereum](https://eips.ethereum.org/EIPS/eip-4361)
- [Ethereum Attestation Service](https://attest.org/)

### Stellar

- [CAP-51: Smart Contract Host Functionality, secp256r1 Verification](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0051.md)
- [SEP-10: Stellar Web Authentication](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0010.md)
- [SEP-12: KYC API](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0012.md)
- [SEP-30: Account Recovery](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0030.md)
- [Stellar Smart Wallets](https://developers.stellar.org/docs/build/guides/contract-accounts/smart-wallets)

### Regulatory

- [FATF Guidance for a Risk-Based Approach to Virtual Assets and VASPs](https://www.fatf-gafi.org/en/publications/Fatfrecommendations/Guidance-rba-virtual-assets-2021.html)
- [FATF 2025 Targeted Update on Implementation of the Standards on Virtual Assets/VASPs](https://www.fatf-gafi.org/en/publications/Fatfrecommendations/targeted-update-virtual-assets-vasps-2025.html)
