# RFC: Forge Network Service Identities

Status: Experimental

## Authors

- [Alan Shaw](https://github.com/alanshaw)

## Introduction

Proposal for service identities/URLs in the Forge network.

## Proposal

### Staging

- `did:web:staging.upload.fil-forge.com` (Sprue)
- `did:web:staging.indexer.fil-forge.com` (Indexing Service)
- `did:web:staging.signer.fil-forge.com` (Signing Service)
- `did:web:staging.delegator.fil-forge.com` (Delegator)
- `did:web:staging.auth.fil-forge.com` (Hilt)
- `did:web:staging.revoke.fil-forge.com` (Swarf)

We run these services in staging but typically they are run by a 3rd party:

- `did:web:staging.eu-central-1.s3.fil-forge.com` (Ingot)
- `did:key:z...` `https://staging.piri-0.fil-forge.com` (Piri)

### Production

- `did:web:upload.fil-forge.com` (Sprue)
- `did:web:indexer.fil-forge.com` (Indexing Service)
- `did:web:signer.fil-forge.com` (Signing Service)
- `did:web:delegator.fil-forge.com` (Delegator)
- `did:web:auth.fil-forge.com` (Hilt)
- `did:web:revoke.fil-forge.com` (Swarf)

Multiple of these run by 3rd parties:

- `did:web:<REGION>.s3.<DOMAIN>` (Ingot)
- `did:key:z...` `https://piri.<DOMAIN>` (Piri)
