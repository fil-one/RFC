# RFC: Forge Network Service Identities

Status: Experimental

## Authors

- [Alan Shaw](https://github.com/alanshaw)

## Introduction

Proposal for service identities/URLs in the Forge network.

## Proposal

### Staging

- `did:web:upload.staging.fil-forge.com` (Sprue)
- `did:web:indexer.staging.fil-forge.com` (Indexing Service)
- `did:web:signer.staging.fil-forge.com` (Signing Service)
- `did:web:delegator.staging.fil-forge.com` (Delegator)
- `did:web:auth.staging.fil-forge.com` (Hilt)
- `did:web:revoke.staging.fil-forge.com` (Swarf)

We run these services in staging but typically they are run by a 3rd party:

- `did:web:eu-central-3.s3.staging.filonecontent.com` (Ingot)
- `did:key:z...` `https://piri-0.staging.fil-forge.com` (Piri)

### Production

- `did:web:upload.fil-forge.com` (Sprue)
- `did:web:indexer.fil-forge.com` (Indexing Service)
- `did:web:signer.fil-forge.com` (Signing Service)
- `did:web:delegator.fil-forge.com` (Delegator)
- `did:web:auth.fil-forge.com` (Hilt)
- `did:web:revoke.fil-forge.com` (Swarf)

Multiple of these run by 3rd parties:

- `did:web:<REGION>.s3.filonecontent.com` (Ingot)
- `did:key:z...` `https://piri.<PROVIDER_DOMAIN>` (Piri)
