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
      "controller": "<the subject DID>",
      "authority": "<the attesting authority DID>"
    }
  ],
  "capabilityInvocation": [
    "<the subject DID>#attestation"
  ],
  "capabilityDelegation": [
    "<the subject DID>#attestation"
  ]
}
```

The `AuthorityAttestation` verification method type indicates that authentication for this DID is performed by a trusted authority, named in the `authority` field.

In theory, this verification method is available to any kind of DID which can put one in its DID document. In practice, this verification method is likely only useful for the narrow cases described in this document.

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

1. The client authors a delegation payload in which the `did:mailto` issues some capability to the client's agent identity, and issues an invocation to the authority asking for it to be signed.
2. The authority computes a an HMAC over `(delegation_payload, iat, exp)` using a key the authority controls, where `delegation_payload` is the canonical (DAG-CBOR) encoding of the delegation payload, `iat` is the time the link was issued, and `exp` is some expiration time for the link. `iat` is optional, but will result in a timestamp on the final signature for tracking purposes.
3. The authority sends an email to the address encoded in the `did:mailto` DID, containing a verification link pointing back to the authority's web server. The URL's params include the delegation payload, the `exp`, and the HMAC, suitably encoded.
4. When the user clicks the link, the authority validates the HMAC.
5. The authority produces an attestation (§4.3) and builds a signed delegation from the payload, using the attestation bytes as the signature. It stores that delegation for later retrieval.

The authority MAY use any format it chooses for the verification link URL. The authority MAY encode both the payload and the HMAC in the URL using any URL-suitable encoding. The authority produces and consumes this URL, so the format is irrelevant to the spec, as long as the authority can recover the values from the URL.

The authority SHOULD include a full description of the delegation payload in the email it sends. For instance, it could show the DAG-JSON representation, or a more human readable layout. This ensures that the controller of the email inbox has an opportunity to understand what they're authorizing.

If the clicked link is not valid, either because the HMAC does not match or because the `exp` time has passed, the authority MUST NOT create an attestation. The authority's web server MAY display a useful message explaining the failure. The authority's web server MAY offer to send a new email with a new link for the same delegation payload.

The attestation operation is idempotent: clicking the same link a second time produces a byte-identical delegation.

The authority MAY define a policy governing what delegations it will verify and attest to, and reject requests which are not permitted by that policy. For instance, the authority may reject delegations without a recent `nbf` ("not before"), or with an `exp` ("expiration") that is `null` or too far in the future. To reject an attestation request, the authority MUST return a failure for the original attestation request invocation. If it does not reject the request, the authority MUST use the delegation payload as given, with no changes.

The verification link should have a reasonably short expiration. 15–60 minutes is RECOMMENDED.

The authority MAY use a digest in place of the full payload in the URL, but this requires it to store the pending payload while waiting for verification. In this configuration, the stored payload can be evicted from the store when the link expires.

The exact invocation to request an email-loop verification is not specified here. It must send the desired delegation payload in its `args`.

### 3.2 `did:oauth`

*This section is non-normative. It is a proof-of-concept illustration to demonstrate that the broader spec is sound and extensible. It is expected that this section will be normatively redefined more carefully in a future RFC, before implementation.*

1. The requesting party initiates an OAuth2 authorisation flow with the identity provider (IdP) named in the `did:oauth` DID, obtaining an authorisation code.
2. The requesting party presents the authorisation code to the authority.
3. The authority exchanges the authorisation code with the IdP for an ID token, verifies it, and extracts the `sub` claim.
4. The authority confirms the `sub` claim and IdP domain match the `did:oauth` DID.
5. The authority produces an attestation (§4.3) and returns it to the requesting party.

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
0x71                       Payload encoding: DAG-CBOR
```

The header is a constant value which signals that the signature should be interpreted as an authority-attestation signature.

### 4.3 Signature Invocation

The signature bytes (the `.0` field of the UCAN envelope) for this type are the DAG-CBOR encoding of a UCAN invocation with a specific shape:

```
// DAG-JSON
[
  {"/": {"bytes": "..."}},
  {
    "h": {"/": {"bytes": "..."}},
    "ucan/inv@1.0.0-rc.1": {
      "iss": "did:example:attestingauthority",
      "sub": "did:example:attestingauthority",
      "cmd": "/ucan/attest/proof",
      "args": {
        "digest": {"/": {"bytes": "..."}}
      },
      "prf": []
      "nonce": {"/": {"bytes": ""}},
      "exp": null,
    }
  }
]
```

* The **`iss`** and **`sub`** are both the authority's DID. As an issuer, the DID must be resolvable to a verification method which can sign the invocation.
* **`aud`** is missing, because the invocation is not addressed to anyone in particular.
* **`cmd`** is `/ucan/attest/proof`.
* **`args`** contains a single field, `digest`, whose value is a multihash digest of the outer delegation's `SigPayload` (the token payload plus the Varsig header). The digest SHOULD be SHA-256, [the only required algorithm in the UCAN cryptosuite](https://github.com/ucan-wg/spec/blob/main/README.md#cryptosuite). Using another algorithm may limit interoperability.
* **`prf`** is empty. This invocation is issued by its subject, with inherent authority and no proofs required.
* **`meta`** is optional, and can be used by the authority to track extra facts about the verification process, for informational purposes. Information stored in `meta` MUST NOT be considered to affect the validity of the signature.
* **`nonce`** is empty. As an assertion of fact, the invocation is inherently idempotent.
* **`exp`** is `null`. As an assertion of fact, the invocation cannot expire: the signature cannot have not happened because time has passed. The delegation's `exp` controls the expiration of the delegation.
* **`iat`** is optional, and for informational purposes only. If it is provided, it MUST be the time that the attestation request was received, not the time that the signature was created. This ensures that verifying the same request twice produces an identical signature and delegation.
* **`cause`** is optional, and for informational purposes only. If present, it MUST be a link to the receipt of the attestation request invocation, which itself MUST list the signature as an effect. This means that the receipt cannot be created until the signature is created, and cannot be returned synchronously during the attestation request. This is a lot of machinery which may not be of any value, making this field especially optional.

This invocation is itself signed in the normal way, by its issuer, the attesting authority.

---

## 5. Verification

When a verifier encounters a delegation with a Varsig header with the algorithm discriminant `0x300001`, it should:

1. **Decode the signature invocation**: Interpret the signature bytes as a canonically encoded UCAN invocation.

1. **Resolve the issuer DID**: Perform the "Resolve" operation on the delegation issuer's DID. For a `did:mailto`, for instance, this means expanding the DID algorithmically into its DID document. Then find a `capabilityDelegation` verification method with the type `AuthorityAttestation` and an `authority` matching the signature invocation's issuer. If none is found, fail.

2. **Validate the signature invocation**: Resolve the authority's DID and validate the authority's signature on the invocation. If the invocation is invalid, fail.

4. **Verify the digest**: Take the digest of the delegation's `SigPayload` using the same algorithm as the value of `.digest` in the invocation's `args`. If the algorithm is not supported, fail. If the computed digest does not match the invocation args, fail.

5. **Succeed**: If the process has not failed yet, the delegation is valid.

If the process fails at any point, the verifier should consider the delegation to have an invalid signature.

---

## 6. Security Considerations

### 6.1 The Trust Gap

This scheme does not eliminate the need for out-of-band trust configuration. The executor must decide whether to trust a given authority for a given DID method and domain or provider. This is unavoidable: no cryptographic scheme can bootstrap trust from an identity that has no keypair. The scheme makes the trust relationship explicit and self-describing rather than implicit.

The trusted authority is designated in the verification method. Therefore, the trust gap is manifested in the algorithm which expands a DID document from a `did:mailto` or similar DID. DIDs of methods with stored, non-algorithmic DID documents can select their own trusted authority.

### 6.2 Authority Compromise

If the authority's keypair is compromised, an attacker can issue attestations for arbitrary subjects. In this case, the authority SHOULD rotate keys, and reflect that in the authority's own DID document. Executors SHOULD notice the updated DID document and no longer validate attestations signed by the compromised key.

### 6.3 Subject Identity Compromise

The attestation is no stronger than the underlying identity:

- **`did:mailto`**: If the email account is compromised, an attacker can complete the email loop and obtain a valid attestation. This is an inherent property of email-based identity.
- **`did:oauth2`**: If the OAuth account is compromised, or the IdP is malicious, an attacker can obtain a valid ID token and thus a valid attestation. The authority has no way to detect this.

### 6.4 Replay

Attestation is an idempotent operation: attesting the same payload always produces the same invocation, so replaying the verification has no meaningful effect. An attestation for one payload cannot be used for a different payload, since the authority's signature covers the full `SigPayload`.

### 6.5 Canonicalization

The payload hash in the attestation bytes MUST be computed over the canonical DAG-CBOR encoding of the delegation payload, consistent with UCAN's canonicalization requirements. This is the same encoding that is signed over in the `SigPayload`, preventing canonicalization attacks.

