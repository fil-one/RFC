# RFC: Should engineering take on Clockwork?

**Status:** Proposal
**Author:** [James Kurz](https://github.com/jameskurz-filecoin)
**Audience:** Fil One engineers
**Date:** 2026-08-19

## TL;DR

Clockwork is a web app I built. It is not a vendor product and we would not
sell it. It is a quote-to-cash portal for Filecoin Foundation: quotes, orders,
partners, invoices, back office.

I would like the engineering team to look at it, watch a demo, and then decide
whether we own it. This RFC is that question. It is not a request to cut over
billing or replace Stripe.

Alan is right that we might be better off buying something. I want that on the
table.

## What Clockwork is

A Next.js app, Postgres (Supabase), Stripe, and its own login (currently
WorkOS). Repo: [fil-one/clockwork](https://github.com/fil-one/clockwork).
Demo: https://clockwork-commerce-demo.netlify.app/ — I will put the password in
the FF 1Password vault.

Who it is for:

- **Us (FF).** Sales, partnerships, finance, anyone who currently chases a deal
  through email and PDFs.
- **Customers and partners, later.** A portal where they can see a quote,
  accept an order, pay an invoice. Not a SKU we sell.

Who would own the code if we say yes: **engineering**. Uptime, bugs, security,
features. That is the expensive part. It does not mean engineering becomes the
people who talk to customers.

It was written by me, with a lot of AI assistance. Treat it as an unreviewed
codebase that happens to have a working demo, not as a finished internal
platform.

Start here, not the `docs/` folder: the README, then the live demo. Skip
`docs/release-candidate-report.md`. That file is not a good introduction.

## Why I built it

Today a Fil One commercial deal is spread across HubSpot, Stripe, Google Docs,
and people. Fine for one-off pay-as-you-go. Painful for annual contracts,
partners, and "which quote became which invoice."

I wanted one place where:

- a quote has line items that map to Fil One products ("100 TB in eu-central-3")
- accepting the quote becomes an order
- the order is what we provision
- invoices and payments attach to that order
- a partner can see the deals they brought

Stripe still charges the card. Fil One still creates the bucket. Clockwork
would be the commercial record in the middle.

## What I am not claiming

- We are not selling Clockwork.
- We are not cutting over existing PAYG customers.
- We are not giving Clockwork permission to write to Stripe, Auth0, or Forge
  until a later, explicit decision.
- WorkOS is not a new company-wide identity system. Clockwork used it because
  that is how I stood the demo up. Fil One users live in Auth0. If we keep
  Clockwork, we either federate Auth0 into it or rip WorkOS out. I have not
  done that work.

## How it would sit next to Fil One

```
Customer / partner browser
        │
        ▼
   Clockwork (quotes, orders, invoices, partner view)
        │  "please provision this SKU for org X"
        ▼
   Fil One API (Auth0, tenants, buckets, usage)
        │
        ▼
   Forge / Ingot (actual storage)
```

Stripe: only one system writes for a given customer. Today that system is Fil
One. It stays that way until we deliberately move a customer.

"Accepted line" just means a line item on an accepted quote. Example:
`object-storage / 100TB / eu-central-3` on the quote has to match a real Fil
One product, or the order stops before anyone provisions.

There is no finished Fil One ↔ Clockwork machine API. Clockwork has an internal
OpenAPI for its own UI. The integration API is the work this RFC would
authorize us to design, not something already shipped.

Existing Fil One users would need to be linked on purpose. Matching on email
is not good enough. New signups later would go through whatever we decide
after the demo. So yes: a one-time migration of existing accounts, and an
ongoing path for new ones. Neither is designed yet.

## What exists today

- A clickable demo of customer, partner, and internal-operator screens.
- A lot of tests in that repo. They prove the demo workflows, not production
  Stripe / Auth0 / Forge.
- GitHub Actions is currently blocked on the org billing limit.

## The real decision

Engineering would be taking on a sizable TypeScript app that I built largely
solo. Benefits: commercial team and partners get a portal; quotes, orders, and
invoices live in one place; we stop reconciling spreadsheets.

Costs: we own bugs, security, Stripe edge cases, and every feature request
from sales. That competes with storage work. Buying Chargebee / Stripe Billing
+ HubSpot might cover most of it with less code. I do not have a vendor bake-off
yet. If the team would rather go that way, that is a valid outcome of this RFC.

I would like:

1. A 45-minute demo for the team. I will schedule it.
2. A follow-up technical walkthrough of the repo for whoever would own it.
3. A yes / no / buy-instead decision after that, not before.

Until then, please do not treat Clockwork as an incoming dependency of POSIX,
Forge, or the console.

## Decisions needed

1. Are we willing to spend a demo plus a tech review on this?
2. After that: own it, kill it, or trial a vendor?
3. If we own it, who? This means on-call and security, not who talks to
   customers.

## References

- [Clockwork repo](https://github.com/fil-one/clockwork)
- [Live demo](https://clockwork-commerce-demo.netlify.app/)
- [README](https://github.com/fil-one/clockwork/blob/main/README.md)
