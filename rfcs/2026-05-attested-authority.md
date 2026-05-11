# RFC: Attested Authority: A Generic UCAN Attestation Scheme

Status: Standard (draft)

## Authors

- [Petra Jaros](https://github.com/Peeja)

# Attested Authority: A Generic UCAN Attestation Scheme

## Status

Draft — for discussion

---

## Abstract

This document proposes a generic scheme for using externally-verified identities as UCAN delegation subjects. Such identities, including email addresses, OAuth-based identities, and others, have no associated keypair under user control, and therefore cannot sign UCAN delegations directly. A trusted authority performs an out-of-band verification (email loop, OAuth exchange, etc.) and produces a cryptographic attestation on behalf of the subject identity. A generic Varsig signature type (`authority-attestation`) is defined that encodes the authority's attestation in place of a conventional asymmetric signature, with the specific verification method encoded in the attestation payload rather than the type. This allows attested identities to appear as `iss` in root UCAN delegations while remaining structurally honest about the nature of the verification performed.

Concrete DID methods for specific identity types (e.g. `did:mailto`, `did:oauth`) are defined as extensions of this scheme.

---

## 1. Motivation

UCAN delegation chains require every issuer to sign with a private key. Many real-world identities (email addresses, OAuth-based social identities, phone numbers, and others) have no keypair under user control. The existing approach uses an invocation to attest to a delegation's correctness, but that invocation is difficult to keep with the delegation it attests to.

This proposal instead defines a generic attested authority scheme such that:

- An externally-verified identity can appear as `iss` in a UCAN delegation
- The "signature" bytes encode a real cryptographic signature from a trusted authority, together with attestation metadata including what verification method was used
- Verifiers can determine the authority's identity from the Varsig header itself, resolve its DID, and verify the signature without out-of-band configuration
- New verification methods (email loop, OAuth, etc.) can be added without defining new Varsig types or DID verification method types



---

## 2. DID Methods for Attested Identities

### 2.1 Common Structure

All DID methods defined under this scheme share the same DID Document structure. A DID Document for an attested identity MUST contain at least one verification method of type `AuthorityAttestation` and MUST NOT contain conventional key material, as the subject has no keypair.

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1"
  ],
  "id": "<the subject DID>",
  "verificationMethod": [
    {
      "id": "<the subject DID>#attestation",
      "type": "AuthorityAttestation",
      "controller": "<the subject DID>"
    }
  ],
  "authentication": [
    "<the subject DID>#attestation"
  ]
}
```

The `AuthorityAttestation` verification method type indicates that authentication for this DID is performed by a trusted authority, with the specific authority and verification method encoded in the Varsig header and attestation bytes respectively (see §4).

### 2.2 Resolution

DID Documents for attested identities are generally not hosted or published by the subject (who has no infrastructure). For the two methods defined here, the DID Documents are constructed synthetically by verifiers from the DID string alone, since its structure is fully determined by the method.

### 2.3 `did:mailto`

#### Syntax

```
did:mailto:<domain>:<local-part>
```

Where `<domain>` and `<local-part>` correspond to the domain and local parts of an RFC 5321 email address. The `@` separator is replaced by `:` to conform to DID syntax.

**Examples:**

```
did:mailto:example.com:alice
did:mailto:university.edu:j.smith
```

---

### 2.4 `did:oauth`

#### Syntax

```
did:oauth:<provider-domain>:<subject-id>
```

Where `<provider-domain>` is the domain of the OAuth identity provider and `<subject-id>` is the `sub` claim from the provider's ID token, percent-encoded if necessary to conform to DID syntax.

**Examples:**

```
did:oauth:accounts.google.com:1234567890
did:oauth:github.com:u:9876543
```

The `sub` claim is used as the identifier rather than any mutable attribute (such as email or username), as it is the stable, provider-assigned identifier for the user.

---

## 3. The Verification Protocol

Before issuing an attestation, the authority MUST perform out-of-band verification appropriate to the DID method of the subject. The specific verification approach is the authority's internal concern; the attestation bytes (§4.3) do not encode which method was used. The executor trusts the authority to have performed appropriate verification for the subject DID method it attests.

### 3.1 `did:mailto`

1. The authority computes the SHA-256 hash of the canonical (DAG-CBOR) encoding of the delegation payload.
2. The authority sends an email to the address encoded in the `did:mailto` DID, containing a verification link whose identifying parameter is that hash.
3. When the user clicks the link, the authority recomputes the hash from the payload and confirms it matches. No stored token lookup is required — the payload hash serves as the token.
4. The authority produces an attestation (§4.2) and returns it to the requesting party.

The attestation operation is idempotent: attesting the same payload hash for the same subject always produces the same statement. The attestation `exp` field (§4.3) bounds how long after the email loop the attestation may be used to construct a valid delegation.

TK: Should we put anything else in the link? Does it need to be presigned with an exp?

### 3.2 `did:oauth`

1. The requesting party initiates an OAuth2 authorisation flow with the identity provider (IdP) named in the `did:oauth` DID, obtaining an authorisation code.
2. The requesting party presents the authorisation code to the authority.
3. The authority exchanges the authorisation code with the IdP for an ID token, verifies it, and extracts the `sub` claim.
4. The authority confirms the `sub` claim and IdP domain match the `did:oauth` DID.
5. The authority produces an attestation (§4.2) and returns it to the requesting party.

The authority MUST NOT include OAuth token material in the attestation bytes.

---

## 4. The `authority-attestation` Varsig Type

### 4.1 Multicodec Code

This scheme uses a code from the Multicodec private use area, which is guaranteed never to be assigned a conflicting meaning by the Multicodec specification:

```
0x300001  (within private use range 0x300000–0x3FFFFF)
```

This code SHOULD be replaced with a registered code if the scheme is standardised.

### 4.2 Varsig Header Structure

A Varsig header for this type has the following structure:

```
0x34                       Varsig prefix
0x01                       Varsig version 1
0x300001                   authority-attestation algorithm discriminant (varint)
<authority-did-length>     unsigned varint: byte length of the authority DID string
<authority-did-bytes>      UTF-8 encoded authority DID (e.g. "did:key:zAuth...")
0x71                       Payload encoding: DAG-CBOR
```

The authority DID is encoded inline in the header so that verifiers can locate the correct key without any out-of-band configuration. Since the Varsig header is included in the signed payload (per the Varsig recommendation), the authority DID is itself covered by the attestation signature.


### 4.3 Signature Bytes

The signature bytes (the `.0` field of the UCAN envelope) for this type are a DAG-CBOR encoded map:

```
{
  "payload": {
    "subject":      <string: the attested subject DID>,
    "payload_hash": <bytes: SHA-256 hash of the canonical DAG-CBOR encoding of the delegation payload>,
    "timestamp":    <integer: Unix timestamp of successful verification>,
    "exp":          <integer: Unix timestamp after which this attestation is no longer valid>,
    "alg":          <bytes: authority's Varsig header for its own key type, e.g. Ed25519+DAG-CBOR>
  },
  "sig": <bytes: authority's raw signature over the canonical DAG-CBOR encoding of "payload">
}
```

### 4.4 What the Authority Signs

The authority signs the canonical DAG-CBOR encoding of the `payload` field. The verifier extracts `payload`, re-encodes it canonically as DAG-CBOR, resolves the authority DID to obtain its public key, and verifies `sig` against that encoding.

The `payload_hash` field binds the attestation to the specific delegation. The `alg` field identifies the authority's own signature algorithm and is included in the signed payload so that neither it nor the authority's key type can be substituted after the fact.


---

## 5. UCAN Delegation Structure

A root delegation issued under this scheme looks as follows. Note that `iss` is the attested subject DID — this is the structural goal of the scheme.

**Email identity:**

```json
{
  "iss": "did:mailto:example.com:alice",
  "aud": "did:key:zInvoker...",
  "sub": "did:mailto:example.com:alice",
  "cmd": "/widget/crank",
  "pol": [],
  "nonce": "<random bytes>",
  "exp": 1234567890
}
```

**OAuth identity:**

```json
{
  "iss": "did:oauth:accounts.google.com:1234567890",
  "aud": "did:key:zInvoker...",
  "sub": "did:oauth:accounts.google.com:1234567890",
  "cmd": "/widget/crank",
  "pol": [],
  "nonce": "<random bytes>",
  "exp": 1234567890
}
```

In both cases, the UCAN envelope `.0` field contains the `authority-attestation` signature bytes (§4.3) rather than a conventional asymmetric signature. The Varsig header in `.1.h` encodes the `authority-attestation` type and the authority DID.

---

## 6. Proof Chain

A complete invocation proof chain using this scheme (shown for `did:mailto`; the structure is identical for other attested DID methods):

```
Delegation 1  (root)
  iss: did:mailto:example.com:alice
  aud: did:key:zInvoker...
  sub: did:mailto:example.com:alice
  cmd: /widget/crank
  sig: authority-attestation bytes (authority: did:key:zAuth..., method: "email-loop")

Invocation
  iss: did:key:zInvoker...
  aud: did:key:zExecutor...
  sub: did:mailto:example.com:alice
  cmd: /widget/crank
  prf: [CID of Delegation 1]
```

The chain satisfies the UCAN proof chain requirement that `prf[0].iss == sub` (both are `did:mailto:example.com:alice`) and `prf[0].aud == invocation.iss` (`did:key:zInvoker...`).

---

## 7. Verification

Upon receiving an invocation, the executor MUST:

1. **Standard UCAN chain validation**: verify principal alignment, time bounds, and command attenuation as specified in the UCAN Delegation and Invocation specs.

2. **Detect the attestation type**: inspect the Varsig header of the root delegation. If the algorithm discriminant is `0x300001`, proceed with `authority-attestation` verification.

3. **Extract the authority DID** from the Varsig header.

4. **Trust policy check**: determine whether the authority DID is trusted to attest for the subject DID's method and domain or provider. This is a local policy decision. Executors SHOULD maintain an explicit allowlist of trusted authorities per DID method and domain.

5. **Resolve the authority DID** to obtain its public key.

6. **Verify the authority's signature** in the attestation bytes against the delegation's `SigPayload`.

7. **Verify the payload hash** in the attestation bytes matches the SHA-256 of the canonical DAG-CBOR encoding of the delegation payload.

8. **Verify the attestation timestamp and expiry** are within acceptable bounds.

Steps 2–7 are application-defined logic. The executor MUST NOT accept the invocation if any step fails.

---

## 8. Security Considerations

### 8.1 The Trust Gap

This scheme does not eliminate the need for out-of-band trust configuration. The executor must decide whether to trust a given authority for a given DID method and domain or provider. This is unavoidable: no cryptographic scheme can bootstrap trust from an identity that has no keypair. The scheme makes the trust relationship explicit and self-describing (the authority DID is in the Varsig header) rather than implicit.

### 8.2 Authority Compromise

If the authority's keypair is compromised, an attacker can issue attestations for arbitrary subjects. Executors SHOULD support revocation of authority trust, and authorities SHOULD use short-lived attestations (small `exp` windows).

### 8.3 Subject Identity Compromise

The attestation is no stronger than the underlying identity:

- **`email-loop`**: if the email account is compromised, an attacker can complete the email loop and obtain a valid attestation. This is an inherent property of email-based identity.
- **`oauth2`**: if the OAuth account is compromised, or the IdP is malicious, an attacker can obtain a valid ID token and thus a valid attestation. The authority has no way to detect this.

### 8.4 Replay

Attestation is an idempotent operation: attesting the same payload hash for the same subject always produces the same statement, so replaying an attestation has no meaningful effect. An attestation for one payload cannot be used for a different payload, since the authority's signature covers the full `SigPayload`. Replay protection at the invocation level is provided by the UCAN `nonce` field per the standard UCAN spec. The attestation `exp` field limits the window within which a completed verification can be used to construct a delegation.

### 8.5 Canonicalization

The payload hash in the attestation bytes MUST be computed over the canonical DAG-CBOR encoding of the delegation payload, consistent with UCAN's canonicalization requirements. This is the same encoding that is signed over in the `SigPayload`, preventing canonicalization attacks.

---

## 9. Open Questions

- The `AuthorityAttestation` verification method type defined here is not exclusive to `did:mailto` and `did:oauth`. Any DID document — including those resolved via conventional means such as `did:web` — MAY include an `AuthorityAttestation` verification method, indicating that authentication for that DID is delegated to a trusted authority. The implications of this for DID document publishing and authority discovery are not yet specified.
- Should the attestation bytes include the authority's full Varsig header (`alg`), or is the signature algorithm implied by the authority DID's key type? Including it is more explicit but adds bytes.
- Should `did:oauth` encode the IdP as a DID (e.g. `did:web:accounts.google.com`) rather than a bare domain, for consistency with the rest of the DID ecosystem?