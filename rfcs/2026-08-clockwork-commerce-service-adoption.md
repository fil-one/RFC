# RFC: Adopt Clockwork as Fil One's commerce service

**Status:** Proposal
**Author:** [James Kurz](https://github.com/jameskurz-filecoin)
**Date:** 2026-08-18

## TL;DR

I propose that Fil One adopt
[Clockwork](https://github.com/fil-one/clockwork) as a standalone service for
commerce and commercial operations.

Clockwork would own the customer, partner, and operator workflows around
agreements, quotes, POCs, orders, billing, renewals, and offboarding. The
existing Fil One product would continue to own storage resources, raw usage,
and provisioning. Billing authority would move only through an explicit
account/domain cutover. We would connect the two through a small, explicit API
and event boundary rather than merge Clockwork into the Fil One monorepo.

Merging this RFC would approve the direction and an integration phase. It would
not authorize production migration or billing cutover.

## Why

Fil One's commercial workflow crosses product code, Stripe, HubSpot, provider
dashboards, documents, and manual coordination. That works for individual
transactions but makes it hard to answer basic operational questions reliably:

- Who owns the customer and organization identity?
- Which quote, order, entitlement, invoice, and payment belong together?
- Who is allowed to change pricing or approve an exception?
- Which system should write to Stripe, provisioning, or CRM?
- How do we reconcile and recover after a partial failure?

Clockwork was built to make those relationships explicit. It provides one
customer, partner, and internal-operator product around the same commercial
records, with idempotent workflows, audit history, generated documents, and
provider boundaries.

The implementation now exists and can be evaluated as a product. The remaining
question is whether we want it to become part of Fil One and invest in the real
integration.

## Proposal

1. **Adopt Clockwork as an adjacent service.** Keep its repository, database,
   deployment, and release lifecycle separate from the storage control plane.
2. **Keep one authority for each fact and side effect.** Existing Fil One
   systems remain authoritative until a specific domain and account are
   deliberately cut over. We do not dual-write Stripe, usage, provisioning, or
   CRM.
3. **Reuse existing identifiers.** Clockwork attaches Fil One organization,
   tenant, Stripe, entitlement, and provider IDs. It does not recreate an
   organization because an email or domain happens to match.
4. **Integrate through signed, versioned, idempotent messages.** Transport
   success is not business success; both services retain enough information to
   retry and reconcile.
5. **Roll out incrementally.** First observe, then shadow, then move one
   authority for a small cohort. Existing PAYG billing can remain on its current
   path until we have evidence that moving it is worthwhile and safe.

## Service boundary

| Area | Proposed authority |
| --- | --- |
| Authentication | Fil One Auth0 remains the identity authority for existing users. Clockwork receives a verified subject mapping; email matching is not authorization. |
| Organizations and tenants | Fil One organization and tenant IDs remain the product identity. Clockwork owns the related commercial account and roles. |
| Products, resources, and raw usage | Fil One owns provisionable capability, storage resources, entitlements, and raw measurements. |
| Pricing, agreements, quotes, and orders | Clockwork owns approved commercial configuration and workflow. Accepted lines map to versioned Fil One product capabilities. |
| Stripe and billing | The current integration remains authoritative at first. Exactly one service writes each customer, subscription, meter, invoice, or payment domain after an explicit cutover. |
| Provisioning | Clockwork sends an approved command; Fil One owns resource state and returns accepted and terminal outcomes. |
| HubSpot | Existing integrations keep their current fields until field-level ownership is agreed. Clockwork publishes only allow-listed commercial events. |
| Support | The support provider owns tickets. Clockwork reads safe support signals unless a separate write boundary is approved. |
| Audit | Each service keeps its own domain audit history and links records with correlation and provider IDs. |
| Migration | Fil One remains source authority until a per-account cutover. Clockwork owns migration-run, mapping, and projection evidence. |

The field-level design is already documented in Clockwork's
[adjacent-service integration boundary](https://github.com/fil-one/clockwork/blob/55b4082d380e086b29da7e76f4e060d19cbb49a6/docs/adjacent-service-integration.md).

## Integration sequence

### 1. Connect identity and organizations

Map Auth0 issuer and subject to the Clockwork user, and map each Fil One
organization to a Clockwork account. Missing or ambiguous mappings fail closed.
We should decide whether this is federation or an explicit account-link step
after reviewing the current Auth0 setup.

### 2. Shadow the existing product

Import a pinned snapshot and consume real signed events without allowing
Clockwork to write to Stripe, usage, provisioning, or HubSpot. Compare users,
organizations, products, customers, subscriptions, usage, entitlements,
invoices, and provider objects.

### 3. Pilot one bounded path

Start with a small non-production cohort and one commercial path. My suggested
first candidate is a net-new annual or partner-assisted order, because it avoids
moving an existing PAYG subscription. Rehearse retries, duplicates,
out-of-order events, reconciliation, and rollback before enabling the provider
effect.

### 4. Expand by domain

Move additional accounts or authority domains only after the prior cohort has
zero unexplained money, usage, entitlement, or tenant variance. A rollback
restores the previous routing owner and replays retained events; it does not
delete provider or audit facts.

## What exists today

The reviewed revision is
[`55b4082`](https://github.com/fil-one/clockwork/tree/55b4082d380e086b29da7e76f4e060d19cbb49a6).

- The [live guided demo](https://clockwork-commerce-demo.netlify.app/) exercises
  customer, partner, and internal-operator workflows with resettable demo data.
- The serial demo suite passed 15 of 15 Playwright checks, including
  quote-to-order,
  price-book approval, queue refresh, partner work, customer settings, and
  sandbox payment.
- The web suite passed 1,659 unit tests before two deployment follow-ups; the
  follow-ups passed focused proxy, payment, and hydration tests. The database
  gate exercised 45 pgTAP files and 896 assertions.
- Formatting, lint, types, generated-contract checks, traceability, security
  checks, and the production build passed locally. A separate remote smoke
  passed against the live Netlify deployment.
- GitHub Actions did not start because of the organization's current
  billing/spend limit. That needs to be restored before hosted checks can be a
  production requirement.

The demo is evidence of the product and deterministic workflows, not evidence
that real providers are active. Production identity mapping, machine
authentication, provider accounts, reconciliation, and migration are not wired
yet. They are the work proposed by this RFC.

## Before production

The integration should not launch until:

- identity and organization mapping is unambiguous;
- the service API has explicit machine authentication;
- there is one verified writer for each external effect;
- cross-tenant, duplicate, delayed, reordered, and failed-message cases pass
  across the real Fil One boundary;
- cutover and rollback have been rehearsed for the pilot cohort;
- money, usage, entitlement, and provider objects reconcile with no unexplained
  variance; and
- normal service basics are in place: an owning team, production hosting,
  telemetry/on-call, backups, repository protections, and production-use
  licensing.

## Alternatives

### Put the workflows in the Fil One monorepo

This reduces one deployment but couples a large commercial domain and its
providers to the storage control plane. It also makes the migration authority
boundary less clear. I do not think the trade is worthwhile.

### Continue with the current distributed process

This avoids integration work but leaves the customer journey and reconciliation
spread across systems and manual process. It does not solve the problem
Clockwork was built for.

### Buy a commerce platform

We could run a focused comparison if the team believes a vendor covers the
whole scope. We would still need Fil One-specific identity, usage, entitlement,
provisioning, and reconciliation integration, so feature overlap alone is not a
replacement for this boundary.

## Decisions needed

1. Do we want Clockwork to become Fil One's commerce service?
2. Which team should own it after adoption?

Those are the decisions needed to accept the direction. The integration phase
can then settle whether identity uses federation or account linking, which
bounded path pilots first, and where the production service runs.

## References

- [Clockwork repository](https://github.com/fil-one/clockwork)
- [Reviewed Clockwork revision](https://github.com/fil-one/clockwork/tree/55b4082d380e086b29da7e76f4e060d19cbb49a6)
- [Live demo](https://clockwork-commerce-demo.netlify.app/)
- [Product specification](https://github.com/fil-one/clockwork/blob/55b4082d380e086b29da7e76f4e060d19cbb49a6/commerce_platform_spec.md)
- [Adjacent-service integration boundary](https://github.com/fil-one/clockwork/blob/55b4082d380e086b29da7e76f4e060d19cbb49a6/docs/adjacent-service-integration.md)
- [External activation gates](https://github.com/fil-one/clockwork/blob/55b4082d380e086b29da7e76f4e060d19cbb49a6/docs/external-gates.md)
