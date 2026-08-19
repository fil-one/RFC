# RFC: Should engineering take on Clockwork?

**Status:** Proposal
**Author:** [James Kurz](https://github.com/jameskurz-filecoin)
**Audience:** Fil One engineers
**Date:** 2026-08-19

## TL;DR

Clockwork is a quote-to-cash web app I built for Filecoin Foundation. We would
not sell it. Off-the-shelf is allowed to win.

I would like the team to click through the demo, then decide: **own it, kill
it, or trial a vendor.** This RFC is that question. It is not a billing
cutover, not a Fil One API design, and not a dependency of POSIX or the
console.

## What Clockwork is

A TypeScript monorepo I built, with a lot of AI assistance. Repo:
[fil-one/clockwork](https://github.com/fil-one/clockwork).

Who it is for:

- **Us (FF), first.** Sales, partnerships, finance — anyone who currently
  chases a deal through email, PDFs, HubSpot, and Stripe.
- **Customers and partners, later.** A portal where they can see a quote,
  accept an order, pay an invoice. Not a SKU we sell.

Sales and finance have not agreed they will use this. That is part of what
the demo is for.

Who would own the code if we say yes: **engineering**. Uptime, bugs, security,
features. Not who talks to customers.

Treat it as an unreviewed codebase with a working demo, not a finished
internal platform.

Start at the README, then the live demo. Skip `docs/`, especially
`docs/release-candidate-report.md`. The README line “one artifact chain, no
re-keying” is brochure language. Ignore it.

## What you can click today

Hosted demo: https://clockwork-commerce-demo.netlify.app/

That site is Next.js on Netlify with fake data in Netlify Blobs. It does
**not** run Postgres, Stripe, or WorkOS. There is no production on-call.

The gate is a shared password, then a persona picker. Password lives in the
Netlify env (`CLOCKWORK_DEMO_ACCESS_PASSWORD`). I will put it in the FF
1Password vault. It is not in the git repo.

Flagship path: `/demo` → **Start as Mara Voss** → follow **Review the issued
version and proceed to acceptance** on quote `Q-2026-0312` → accept the order
with PO `PO-DEMO-0312` → confirm the PDF → **Restore demo data** when you are
done (draft and prod share the same blob store).

## Login, Stripe, Fil One — as they actually are

I read the code. Three different systems:

| Who | Login | What they touch |
| --- | --- | --- |
| Fil One customers | Auth0, Fil One console | Buckets, usage, today’s PAYG Stripe |
| Clockwork **demo** visitors | Shared password + persona cookie | Fake quotes/orders. No WorkOS. No Stripe. |
| Clockwork **if we ran it for real** | WorkOS AuthKit (ADR 0004) | Clockwork’s own DB. Stripe SDK exists (`StripeFinanceGateway`) but is not the writer for Fil One customers. |

Clockwork does not sit in front of Auth0. Fil One does not proxy Clockwork.
People who buy storage still log into Fil One. People who issue quotes would
log into Clockwork.

If we keep Clockwork, we either federate Auth0 into WorkOS or rip WorkOS out.
That work is not done.

Stripe: only one system writes for a given customer. Today that is Fil One. It
stays that way until we deliberately move a customer. The demo never calls
Stripe.

There is no Fil One ↔ Clockwork machine API. There are HTTP ports and fakes
pointing at unnamed URLs. The integration API is not in scope of this RFC. Do
not block POSIX or the console on it.

## Why I built it

A Fil One commercial deal is spread across HubSpot, Stripe, Google Docs, and
people. Fine for one-off PAYG. Painful for annual contracts, partners, and
“which quote became which invoice.”

I wanted one place where a quote’s line items map to Fil One products
(“100 TB in eu-central-3”), accepting the quote becomes an order, and invoices
attach to that order.

That is a product wish, not a reason we have to own this repo.
Chargebee, or Stripe Billing + HubSpot, might cover most of it with less code.
I have not done that comparison. A vendor is allowed to win.

## What I am not claiming

- We are not selling Clockwork.
- We are not cutting over existing PAYG customers.
- We are not asking Fil One to proxy Auth0 or Stripe through Clockwork.
- We are not asking anyone to own this on the basis of this document alone.

## The real decision

Engineering would be taking on Next.js, Supabase, WorkOS, a Stripe adapter,
Trigger.dev, and every sales feature request. That competes with storage work.

I would like:

1. A 45-minute demo. I will schedule it.
2. A follow-up walkthrough of the repo for whoever would own it.
3. A yes / no / **trial a vendor** decision after that.

Until then, please do not treat Clockwork as an incoming dependency of POSIX,
Forge, or the console.

## Decisions needed

1. Are we willing to spend a demo plus a tech review on this?
2. After that: own it, kill it, or trial a vendor?
3. If we own it, who is on-call?

## References

- [Clockwork repo](https://github.com/fil-one/clockwork)
- [Live demo](https://clockwork-commerce-demo.netlify.app/)
- [README rewrite PR](https://github.com/fil-one/clockwork/pull/35)
