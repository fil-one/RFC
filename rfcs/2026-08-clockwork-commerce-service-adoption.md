# RFC: Use Clockwork to unblock the quote-to-order journey we show clients

**Status:** Proposal
**Author:** [James Kurz](https://github.com/jameskurz-filecoin)
**Audience:** Fil One engineers
**Date:** 2026-08-19

## TL;DR

Enterprise and channel conversations still cannot walk one path from a tailored
quote to approval, order, provisioning, and billing. That work lives in HubSpot,
Stripe, PDFs, and people. It makes Fil One look less finished than the storage
product, and it blocks deals we should be able to close or demo.

[Clockwork](https://github.com/fil-one/clockwork) is a working prototype of that
journey. We would not sell it as a product. This RFC asks whether the customer
value is worth a **real pilot** — and if so, whether we own this app, or buy
something that covers the same path.

Off-the-shelf is allowed to win. Doing nothing is not, if we keep promising
partners a coherent commercial experience we cannot show.

**Decision after a 45-minute demo:** own a pilot, trial a vendor, or kill it.
Not a billing cutover, not a Fil One rewrite.

## Who is stuck today

- **A prospective customer** who needs an annual or custom quote, not self-serve
  PAYG. Today that is email and a PDF. They cannot see the quote, accept it, or
  watch it become an account.
- **A channel partner** who should register a deal, price it, and see commission
  without a spreadsheet.
- **Us (sales, partnerships, finance)** who cannot point at one record and say
  which quote became which order, which invoice, which tenant.

PAYG in the Fil One console is fine. This is everything next to it.

Clockwork is for all three audiences. The hosted demo lets you click each of
them. Sales and finance have not signed off that they will live in it; the demo
is how we find that out.

## What you can show a client tomorrow

https://clockwork-commerce-demo.netlify.app/

Shared password (Netlify env; I will put it in the FF 1Password vault before
we sit down — it is not in git). Then pick a persona. The site is fake data on
Netlify. It does not talk to Stripe, WorkOS, Auth0, or Forge.

Walk this:

1. `/demo` → **Start as Mara Voss** (customer).
2. Open the issued annual quote and accept it (PO `PO-DEMO-0312`).
3. Download the order-form PDF. The quote flips to accepted.
4. Switch to an internal persona if you want the operator view.
5. **Restore demo data** when you are done.

That is the impression I want in the room: quote → accept → order, in one
place, looking like a product. The RFC is whether we make a slice of that
true against Fil One.

## What Clockwork is (and is not)

A TypeScript app I built, with a lot of AI assistance. Not a vendor. Not a
SKU we sell.

The **demo** is Next.js on Netlify plus a password and persona cookies.

**If we ran it for real:** Next.js on Vercel, Supabase Postgres, WorkOS AuthKit
for Clockwork login, a Stripe adapter in-repo, Trigger.dev. Fil One customers
would still log into Fil One with Auth0. Clockwork would not sit in front of
Auth0 or become the PAYG Stripe writer.

There is no Fil One API yet. Provisioning is typed ports and fakes. A pilot
would need the smallest real call: “this approved order means enable this
product for this org.” That is later work. This RFC does not design it.

Engineering would own uptime, bugs, and security if we keep the app. That
competes with storage work. Name that cost before we fall in love with the
demo.

## Alternatives

- **Pilot Clockwork.** My lean, because we can put it in front of a client
  now. Smallest next step after the demo: one non-PAYG path, no Stripe cutover,
  no Auth0 federation.
- **Buy Chargebee, or Stripe Billing + HubSpot.** Allowed to win. I have not
  done a bake-off. If the team would rather buy, that is a valid result of
  this RFC — as long as we can still demo a coherent journey to clients.
- **Keep stitching email/PDF/HubSpot.** Cheap for engineering. Continues to
  block the conversations that need a quote-to-order story.

## What I am not asking for

- Selling Clockwork.
- Moving existing PAYG customers off Fil One Stripe.
- POSIX, Forge, or console work waiting on this.
- Anyone owning the repo on the basis of this document alone.

## Decisions needed

1. Sit through the demo.
2. Then: **pilot Clockwork**, **trial a vendor**, or **kill it**.
3. If we pilot: who is on-call, and what is the one customer path in scope.

## References

- [Live demo](https://clockwork-commerce-demo.netlify.app/)
- [Clockwork repo](https://github.com/fil-one/clockwork)
- [README rewrite (not merged)](https://github.com/fil-one/clockwork/pull/35)
