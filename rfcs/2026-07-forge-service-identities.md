# RFC: Forge Network Service Identities

Status: Experimental

## Authors

- [Alan Shaw](https://github.com/alanshaw)

## Introduction

Proposal for service identities/URLs in the Forge network.

## Proposal

Piri hostnames have one shape in every stage: `piri-<N>.<REGION>.` followed by
the stage's Forge domain or, in production, the provider's own. An appliance
runs one Piri per storage volume, numbered from 0 within the appliance. The
region label, which Ingot's hostname also carries, keeps the names of Piris in
different regions apart and shows which region a Piri serves.

### Staging

- `did:web:upload.staging.fil-forge.com` (Sprue)
- `did:web:indexer.staging.fil-forge.com` (Indexing Service)
- `did:web:signer.staging.fil-forge.com` (Signing Service)
- `did:web:delegator.staging.fil-forge.com` (Delegator)
- `did:web:auth.staging.fil-forge.com` (Hilt)
- `did:web:revoke.staging.fil-forge.com` (Swarf)

We run these services in staging but typically they are run by a 3rd party:

- `did:web:s3.eu-central-3.staging.filonecontent.com` (Ingot)
- `did:key:z...` `https://piri-<N>.eu-central-3.staging.fil-forge.com` (Piri)

### Dev

Ephemeral and personal stages (PR previews, developer sandboxes) live under a
single `dev` label, so all of them share one DNS delegation and a stage can be
created or torn down without touching the parent zone. `<STAGE>` is a personal
handle (e.g. `bajtos`) or a PR number (e.g. `pr-920`).

- `did:web:upload.<STAGE>.dev.fil-forge.com` (Sprue)
- `did:web:indexer.<STAGE>.dev.fil-forge.com` (Indexing Service)
- `did:web:signer.<STAGE>.dev.fil-forge.com` (Signing Service)
- `did:web:delegator.<STAGE>.dev.fil-forge.com` (Delegator)
- `did:web:auth.<STAGE>.dev.fil-forge.com` (Hilt)
- `did:web:revoke.<STAGE>.dev.fil-forge.com` (Swarf)
- `did:web:s3.<REGION>.<STAGE>.dev.filonecontent.com` (Ingot)
- `did:key:z...` `https://piri-<N>.<REGION>.<STAGE>.dev.fil-forge.com` (Piri)

### Production

- `did:web:upload.fil-forge.com` (Sprue)
- `did:web:indexer.fil-forge.com` (Indexing Service)
- `did:web:signer.fil-forge.com` (Signing Service)
- `did:web:delegator.fil-forge.com` (Delegator)
- `did:web:auth.fil-forge.com` (Hilt)
- `did:web:revoke.fil-forge.com` (Swarf)

Multiple of these run by 3rd parties:

- `did:web:s3.<REGION>.filonecontent.com` (Ingot)
- `did:key:z...` `https://piri-<N>.<REGION>.<PROVIDER_DOMAIN>` (Piri)
