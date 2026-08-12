# RFC: Account Identity and Authorization

Status: Experimental

Last Modified: 2026-04-30

## Authors

- [Petra Jaros](https://github.com/Peeja)

## Motivation

Accounts in Forge are identified by `did:mailto` DIDs. Because email addresses have no associated cryptographic keys, account-rooted UCAN delegations cannot carry meaningful signatures. The current implementation works around this with two coupled mechanisms:

1. **Dummy-signed delegations**: account-to-agent delegations carry placeholder signatures that verifiers ignore.
2. **Session attestations**: a trusted oracle issues `ucan/attest` capabilities that vouch for the dummy-signed delegations after the user confirms via an out-of-band email loop. (See [`w3-session`](https://github.com/storacha/specs/blob/main/w3-session.md).)

This pattern works but has structural problems worth exploring alternatives for:

- **Verifier complexity**: Verifiers must implement a special-case rule — ignore the signature on `did:mailto` delegations and look up an accompanying attestation. This rule is specific to one kind of issuer and inconsistent with how UCAN delegations are otherwise validated.
- **Unstated trust dependency**: The trust in the oracle is system-wide and implicit. Every verifier must be configured to trust attestations from a particular service. This is workable for a single deployment but does not generalize cleanly to multi-issuer or external-identity-provider scenarios.
- **No support for long-lived authorizations**: A delegation rooted at an account DID is valid only as long as the corresponding attestation is. For session-scoped delegations this is fine, but for delegations intended to outlive a session, the attestation lifetime becomes a limit on delegation lifetime. For example, a delegation to a content-serving gateway might try to remain valid for months, but if the session that created it expires or is revoked, it dies with it.
- **Limited extensibility to other identity providers**: The current scheme is shaped specifically around email loops. Extending to OIDC (Auth0 and similar) requires significant rework.

This RFC explores an alternative design that addresses these problems by reframing the identity model around explicit Identity Providers, expressing key bindings as W3C Verifiable Credentials carried in delegation metadata, and using transparency-log timestamps to decouple long-lived delegation validity from session lifetime.

## Hypothesis / Goals

If we restructure the account identity layer around a small set of standard primitives — Identity Providers (OIDC vocabulary), Verifiable Credentials (W3C VC vocabulary), and transparency logs — we can:

1. Eliminate the dummy-signature pattern. Account-to-agent delegations carry real signatures from real keys, with the key-to-account binding expressed as a verifiable credential.
2. Make trust in any particular Identity Provider explicit at the identifier layer rather than as system-wide configuration. A verifier evaluating an account DID can read which Identity Provider attests to it and decide whether to honor that provider's claims.
3. Support long-lived delegations whose validity is decoupled from session lifetime, by binding the signing event to a transparency-log timestamp falling within the credential's valid window.
4. Generalize naturally to non-email Identity Providers (Auth0, OIDC generally, future identity systems) without special-casing.

The design should preserve the user-experience properties of the current scheme: email-loop login as a baseline, no client-side key management burden beyond what already exists, and no new transports introduced beyond what is already in the stack.

## Design

The design comprises three coordinated components. Each addresses a different concern, but they are designed together because they reinforce each other when adopted as a set.

### Overview

```
           ┌─────────────────────────────────────────┐
           │  Persistent Authorizations              │
           │  (timestamps for long-lived delegations)│
           └─────────────────┬───────────────────────┘
                             │ depends on
           ┌─────────────────▼───────────────────────┐
           │  Embedded Identity Credentials          │
           │  (credentials in UCAN facts)            │
           └─────────────────┬───────────────────────┘
                             │ depends on
           ┌─────────────────▼───────────────────────┐
           │  did:idp Method                         │
           │  (IdP-parameterized account DIDs)       │
           └─────────────────────────────────────────┘
```

A reader can adopt any prefix of this stack; each layer is meaningful on its own.

### Component 1: `did:idp`, an IdP-parameterized DID method

Account DIDs are restructured so that the Identity Provider that vouches for them is named within the identifier itself. The DID method `did:idp` has the form:

```
did:idp:<idp-encoded>:<subject-encoded>
```

Where `<idp-encoded>` is the percent-encoded DID of the Identity Provider, and `<subject-encoded>` is the percent-encoded form of an opaque identifier whose meaning is defined by the Identity Provider. Examples:

```
did:idp:did%3Aweb%3Aauth.fil-one.network:alice%40example.com
did:idp:did%3Aweb%3Aacme.auth0.com:auth0%7C507f1f77bcf86cd799439011
```

The two identifiers above might refer to the same human under different Identity Providers. They are nevertheless distinct DIDs, with no implied relationship — each is what its named Identity Provider says it is.

**Key design properties:**

- **IdP-explicit identity.** Every `did:idp` identifier names the Identity Provider that vouches for it. The trust claim a verifier accepts when honoring a `did:idp` DID is not "the identified principal really controls the underlying account in any external sense" but rather "the named Identity Provider is consistent about which key represents this DID." The latter is verifiable by observation; the former is not.

- **Subject-type independence.** The method does not constrain what kind of identifier the Identity Provider uses for accounts. Email addresses, OIDC subjects, or other formats are all permitted; the Identity Provider's namespace defines what is meaningful. Consequently the method generalizes to non-email IdPs without modification.

- **Document derivation from credentials.** The DID document for a `did:idp` identifier is constructed deterministically from credentials issued by the named Identity Provider for that subject. There is no registry; verifiers evaluate the credentials they have available.

The DID document for `did:idp:<idp>:<subject>` has its `controller` set to the Identity Provider's DID. The `verificationMethod` array is populated from currently-valid credentials issued by the named Identity Provider whose subject matches the DID. Two verifiers may produce different DID documents for the same identifier at the same time, depending on which credentials they have available — this is intentional, and matches the posture of the existing `did:mailto` spec ("the same identifier MAY correspond to different DID documents in different sessions").

### Component 2: Embedded Identity Credentials in UCAN delegations

The binding of a key to an account identity is expressed as a W3C [Verifiable Credential](https://www.w3.org/TR/vc-data-model-2.0/), embedded in the `fct` field of the UCAN delegation it accompanies.

A credential vouching for a session key has the shape:

```json
{
  "@context": ["https://www.w3.org/ns/credentials/v2"],
  "type": ["VerifiableCredential", "VerificationMethodCredential"],
  "issuer": "did:web:auth.fil-one.network",
  "validFrom": "2026-04-30T12:00:00Z",
  "validUntil": "2026-05-30T12:00:00Z",
  "credentialSubject": {
    "id": "did:idp:did%3Aweb%3Aauth.fil-one.network:alice%40example.com",
    "verificationMethod": {
      "id": "did:idp:...#k1",
      "type": "Multikey",
      "publicKeyMultibase": "zABCDEF..."
    }
  },
  "credentialStatus": { /* BitstringStatusListEntry */ },
  "proof": { /* signature by the Identity Provider */ }
}
```

When the user delegates capabilities to an agent, the delegation is signed by a real key (the session key the credential vouches for), with the credential embedded in `fct`:

```json
{
  "iss": "did:idp:did%3Aweb%3Aauth.fil-one.network:alice%40example.com",
  "aud": "did:key:zSession",
  "att": [ /* capabilities */ ],
  "exp": 1717113600,
  "fct": [
    { "credential": { /* the credential as above */ } }
  ],
  "s": /* signature by the credentialed session key */
}
```

A verifier processing this delegation:

1. Locates the credential in `fct`. If absent, rejects.
2. Validates the credential: signature verifies, validity period covers signing time, not revoked, `issuer` is one the verifier accepts for the account DID.
3. For `did:idp` accounts: confirms the credential's `issuer` field matches the Identity Provider DID encoded in the account's identifier.
4. Confirms the credential's `verificationMethod` matches the key that signed the delegation.
5. Verifies the delegation's signature using that key.
6. Proceeds with normal UCAN validation.

This eliminates the dummy-signature pattern. The delegation has a real signature from a key with a real, verifiable claim to act for the account. The Identity Provider's role becomes "credential issuer" — a standard identity-layer function — rather than "UCAN delegate that vouches for opaque sessions."

**Where the signing key originates** is left as an implementation choice. Two patterns are conformant:

- **Client-side ephemeral key**: The agent generates a fresh keypair, transmits its public part to the Identity Provider after the email loop, receives a credential vouching for it, signs the account-to-agent delegation, and discards the keypair. Subsequent invocations are signed by the agent's own key (which the delegation's `aud` identifies). Under this model, the Identity Provider never holds a key authorized to sign as the account.

- **Server-side ephemeral key**: The Identity Provider generates a fresh keypair, signs the account-to-agent delegation, embeds a credential vouching for it, and discards the keypair. Under this model, the agent receives a fully-formed delegation with no client signing required.

Both produce identical artifacts on the wire and are equally verifiable. The first offers stronger non-repudiation; the second is simpler operationally and matches the current `access/claim` flow.

**Transport**: No new transport is required. The existing `access/claim` flow that returns delegations-by-audience can deliver credential-bearing delegations exactly as it currently delivers attestation-bearing ones. The artifact's shape changes; its delivery path does not.

### Component 3: Persistent authorizations via transparency-logged credentials

Embedded credentials have session-scoped lifetimes. For delegations intended to outlive a session — for example, a long-lived delegation to a gateway service authorizing it to serve content — the credential's expiration would invalidate the delegation prematurely.

The mechanism: when minting a long-lived delegation, the user submits both the delegation and the underlying credential to a transparency log and obtains an inclusion proof. The proof attests that the delegation existed in the log at a particular time, signed by a key the credential vouched for at that time. The inclusion proof is embedded in the delegation's `fct` field alongside the credential.

When verifying a long-lived delegation whose credential is no longer current, the verifier checks:

1. The delegation's transparency proof is valid against a recognized log.
2. The credential's transparency proof shows it was valid at the delegation's logged timestamp.
3. The delegation's signature is by the key the credential vouched for.
4. Neither the credential nor the delegation has been revoked.
5. The delegation has not itself expired.

If all checks pass, the delegation is honored — even if the credential has expired in real time. The delegation's validity is anchored to the timestamp at which the log committed to its inclusion, not to the verifier's current time.

This pattern is the standard certificate-authority model applied to UCAN delegations: a brief authentication produces a durable artifact via the issuer's signature, with timestamps establishing when the durable artifact came into existence. It is structurally identical to how TLS certificates remain valid for years after the brief domain-control verification that triggered their issuance.

**Trust implications:**

The Identity Provider is no longer in the chain of authority for long-lived delegations. It issues credentials; the user signs delegations using credentialed keys; the log timestamps the signing event. A compromised Identity Provider can issue rogue credentials going forward, which would be observed in the log. It cannot retroactively manufacture authorizations for past times, since the log's timestamps cannot be backdated. Long-lived delegations issued before any compromise remain trustworthy.

This narrows the Identity Provider's privileged role substantially. Where today the trust dependency is "every verifier must trust the Identity Provider's attestations indefinitely," under this design the trust dependency is "every verifier must trust the Identity Provider's credential-issuance behavior at moments observable in a public log." That dependency is auditable rather than implicit.

### Logout and revocation

Three paths invalidate a long-lived delegation:

- **User-initiated UCAN revocation**: Any current session can sign a UCAN revocation of the delegation. Requires no Identity Provider involvement.
- **Identity-Provider-initiated credential revocation**: The Identity Provider updates its credential status list. This invalidates every delegation depending on the revoked credential. Useful as a backstop for incident response or when a user has lost all current sessions.
- **Natural expiration**: The delegation's own `exp` field is honored normally.

Revocation events SHOULD be logged in the transparency log alongside issuance events, so the Identity Provider's behavior is fully auditable.

### Worked example: gateway authorization

A user authorizes a content-serving gateway to retrieve and serve data from one of their spaces:

1. User logs in via email loop, receiving a session credential.
2. User constructs a delegation:
   - `iss`: `did:idp:...alice@example.com` (their account)
   - `aud`: `did:key:zGateway` (the gateway)
   - `att`: `[{ "with": "<space>", "can": "gateway/serve" }]`
   - `exp`: one year from now
   - `fct`: [the session credential]
3. User signs the delegation with their session key.
4. Client submits the delegation to the transparency log, obtains an inclusion proof.
5. Client embeds the inclusion proof in the delegation's `fct` alongside the credential.
6. Client delivers the finalized delegation to the gateway.
7. Gateway holds the delegation and presents it whenever it serves content from that space.

Six months later, when the user's original session has long since expired:

1. Gateway presents the delegation to a verifier.
2. Verifier checks the credential's transparency proof: the credential was logged at time T₁, valid until T₁ + 30 days.
3. Verifier checks the delegation's transparency proof: the delegation was logged at time T₂, where T₁ ≤ T₂ ≤ T₁ + 30 days.
4. Verifier validates: the credential's signature is good; the delegation's signature is by the key the credential vouched for; neither has been revoked; the delegation hasn't expired.
5. All checks pass: the delegation is honored.

The user retains the ability to revoke the delegation at any time by signing a UCAN revocation with any current session.

## Open Questions

- **Credential proof format.** The W3C VC ecosystem supports multiple proof formats: [Data Integrity](https://www.w3.org/TR/vc-data-integrity/), [JOSE](https://www.w3.org/TR/vc-jose-cose/), and CBOR-aligned options. The right choice depends on alignment with UCAN's existing signature handling and on tooling availability.

- **Transparency log infrastructure.** Should this design use existing log services such as [Sigstore Rekor](https://docs.sigstore.dev/logging/overview/), build a Fil One-operated log, or some combination? Multi-log redundancy (analogous to [Certificate Transparency's SCT requirements](https://certificate.transparency.dev/howctworks/)) is desirable for resilience but adds operational complexity.

- **Migration policy for the existing `ucan/attest` mechanism.** The existing scheme can coexist with the new one for a transition period, with verifiers accepting either form. The right cutover timeline depends on deployment realities.

- **DID method registration.** Should `did:idp` be registered as a [W3C DID method](https://www.w3.org/TR/did-spec-registries/) or treated as Fil One-specific? Registration costs effort but increases interop value if other systems adopt the pattern.

- **Trust framework expression.** How verifiers configure which Identity Providers they accept is left to deployment. Whether Fil One should publish a default trust framework — a list of recommended Identity Providers and credential types — is a separate question worth resolving early.

- **Recovery and key loss.** Under the proposed design, an account holder who loses access to all their devices and email loses the ability to sign UCAN revocations. The Identity Provider can still revoke credentials. Is this an acceptable recovery model, or are additional mechanisms needed (e.g., user-held offline recovery keys)?

- **Subject identifier privacy.** The subject identifier appears in every DID. For email-based accounts this exposes the email address publicly. Identity Providers can mitigate by using opaque subject identifiers (random IDs), at the cost of memorability. The right default warrants design attention.

## Evaluation Criteria

This design will be considered successful if:

- A reference implementation can produce credential-bearing delegations end-to-end (login → credential → signed delegation → verification) without changing the existing `access/claim` transport or the wire format of UCAN invocations beyond the documented `fct`-field additions.

- Verifiers can validate credential-bearing delegations using off-the-shelf VC libraries for the credential portion and existing UCAN libraries for the delegation portion, with the verifier-specific logic limited to bridging the two.

- Long-lived gateway authorizations can be issued, used, and revoked across a one-year window during which the underlying session credentials expire and are renewed multiple times, without the gateway's authorization lapsing prematurely.

- An Auth0 (or other OIDC) integration can be implemented as a parallel Identity Provider issuing credentials of the same shape, with no changes to the verifier or to other system components.

- A user can review their own account's transparency log entries to detect any credential issuance they did not authorize, and a third-party watchdog can do the same systematically across the user base.

Failure modes to watch for during prototyping:

- Excessive credential or transparency-proof size inflating delegation chains beyond practical limits for routine operations.
- Latency penalties on delegation issuance from synchronous transparency-log submission.
- Verifier complexity growth from supporting both old (`ucan/attest`) and new (embedded credential) forms during migration.

## Alternatives Considered

### Continue with `ucan/attest` and adapt incrementally

The simplest alternative is to leave the existing attestation mechanism in place and address the long-lived-delegation problem with longer-lived attestations. This was rejected because it does not address the structural concerns (special-case verifier rules, system-wide implicit trust in the oracle) and creates security tradeoffs (a long-lived attestation gives an attacker a long window of impersonation if the corresponding key is stolen).

### Parameterize `did:mailto` rather than introduce a new method

We could extend `did:mailto` syntax to include the Identity Provider rather than introducing `did:idp`. This was rejected because it would conflict with the published `did:mailto` spec, which has DKIM-based resolution semantics that Fil One does not implement and that are incompatible with the IdP-attested resolution proposed here. A new method name keeps the two cleanly separated.

### Use UCAN delegations rather than W3C VCs for credentials

The credential's role — "this key is a verification method for this DID" — could be expressed as a UCAN delegation rather than a VC. This was considered and rejected: UCAN's design center is capability transfer, while VC's design center is third-party assertion about subjects. The credential is doing assertion work, not capability-transfer work, and forcing it into UCAN vocabulary produces semantically awkward delegations (the existing `ucan/attest` pattern is precisely this). VC vocabulary fits the role.

### Identity-Based Signatures (IBS) instead of credentials

[IBS](https://en.wikipedia.org/wiki/Identity-based_signature) schemes let a trusted authority derive signing keys from arbitrary identifier strings, allowing signatures verifiable using only the identifier and a public master parameter — no per-user credential needed. This is structurally similar to what we want.

It was rejected for several reasons. IBS introduces mandatory key escrow at the trusted authority (the PKG can derive any user's key from the master secret) — strictly worse than the proposed design, where the Identity Provider can issue new credentials going forward but cannot directly produce signatures for keys it doesn't hold. Revocation in IBS is famously difficult compared to standard credential status lists. The library and tooling ecosystem is much smaller than for VCs and JWTs. And IBS does not generalize to external Identity Providers like Auth0, since they don't speak IBS — adopting IBS would foreclose the multi-IdP extension that motivates much of this design.

### Storing credentials externally rather than embedding in UCAN facts

Credentials could be stored at the Identity Provider's endpoint and fetched by verifiers on demand, rather than embedded in delegation `fct` fields. This was rejected because it requires verifiers to be online with the Identity Provider for every verification, which conflicts with UCAN's "everything needed to verify travels with the message" philosophy and creates a liveness dependency on the Identity Provider for routine operations. Embedding credentials in `fct` keeps verification self-contained.

### Relying on existing PKI / TLS certificates

A user's TLS certificate (e.g., a client cert from an existing CA) could in principle serve as evidence of identity. This was rejected because (a) most users don't have client certs for their email addresses, (b) the email-loop authentication flow is the existing UX baseline and changing it is out of scope, and (c) commercial CAs are not generally in the business of attesting to email control in a way useful here.

## References

- [W3C Decentralized Identifiers (DIDs) v1.0](https://www.w3.org/TR/did-1.0/)
- [W3C Verifiable Credentials Data Model 2.0](https://www.w3.org/TR/vc-data-model-2.0/)
- [W3C Bitstring Status List v1.0](https://www.w3.org/TR/vc-bitstring-status-list/)
- [UCAN specification](https://github.com/ucan-wg/spec/)
- [`did:mailto` method specification](https://github.com/storacha/specs/blob/main/did-mailto.md)
- [`w3-session` (storacha)](https://github.com/storacha/specs/blob/main/w3-session.md) — current authorization session mechanism
- [`w3-account` (storacha)](https://github.com/storacha/specs/blob/main/w3-account.md) — current account model
- [`did:web` method specification](https://w3c-ccg.github.io/did-method-web/)
- [`did:key` method specification](https://w3c-ccg.github.io/did-key-spec/)
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html) — `iss + sub` identifier model
- [Certificate Transparency](https://certificate.transparency.dev/) — the transparency-log model this design draws on
- [Sigstore Rekor](https://docs.sigstore.dev/logging/overview/) — a candidate transparency-log implementation
- [transparency.dev](https://transparency.dev/) — general transparency-log infrastructure and patterns