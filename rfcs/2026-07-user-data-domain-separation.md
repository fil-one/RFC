# RFC: User data domain separation

Status: Experimental

## Authors

- [Alan Shaw](https://github.com/alanshaw)

## Motivation

Regional provider S3 endpoints currently live at `<region>.s3.fil.one` — the same registrable domain as our "front of house" properties: the console at `app.fil.one` and the website at `www.fil.one`.

These endpoints serve untrusted, user-controlled content: uploaded objects, presigned URLs, and (in future) potentially browsable/static content. This is a problem because domain reputation systems operate at *registrable domain* granularity, not subdomain:

- **Blocking blast radius.** A single user uploading illegal content or hosting phishing pages via any regional provider can get `fil.one` flagged by Google Safe Browsing, ISP/national DNS blocklists, and corporate proxies. This would take down the console, website, docs and company support email in one shot — for all regions.
- **Independent operators.** Each region is run by an independent entity with its own abuse handling and response times. Fil One cannot guarantee how quickly any given region remediates abuse, yet today all regions and Fil One itself share fate through the one domain.
- **Email deliverability.** Spam/abuse reputation attached to `fil.one` degrades deliverability of company and transactional email sent from the domain.
- **Brand trust.** Phishing served from `*.s3.fil.one` appears endorsed by Fil One.
- **Browser origin hygiene.** Keeping user content off the registrable domain of the console eliminates any class of cookie-scoping or origin mistakes between `app.fil.one` and user-served content.

This is a well-established pattern: AWS serves user data from `amazonaws.com`, not `aws.amazon.com`; GitHub uses `githubusercontent.com`; Google uses `googleusercontent.com`; Cloudflare R2 uses `r2.dev`.

## Hypothesis / Goals

- Abuse originating from user data in **any** region can never impact `fil.one` front-of-house properties (console, website, email).
- Blocking, when it happens, is scoped as narrowly as possible — ideally per-subdomain rather than the whole data domain.
- The move happens **before GA**, while endpoint URLs have minimal embedding in customer configuration, SDKs and documentation.

## Design

1. **Register a dedicated data domain** — proposed: `filone.com`. Regional S3 endpoints become `<region>.s3.filone.com` (e.g. `eu-west-1.s3.filone.com`).
2. **Nothing but user data lives on this domain.** No console, no marketing pages, no email. Publish a null MX and restrictive SPF/DMARC records so the domain cannot be used to send mail.
3. **Add the domain to the [Public Suffix List](https://publicsuffix.org/).** With `filone.com` (or `s3.filone.com`) listed as a public suffix, browsers treat each subdomain as an independent site — cookies cannot be scoped across subdomains, and reputation systems that respect the PSL (including Safe Browsing) can flag an individual subdomain rather than the whole domain.
4. **Migration.** Presigned URLs sign the `Host` header, so existing `*.s3.fil.one` URLs cannot be redirected (a redirect changes the host and breaks the SigV4 signature). Regional providers serve **both** domains during a deprecation window at least as long as the maximum presigned URL expiry, with `fil.one` endpoints removed thereafter. Console, docs and SDK-generated endpoint URLs switch to the new domain immediately.

## Alternatives Considered

- **Status quo, rely on abuse takedowns.** Reactive only — a Safe Browsing flag or blocklist entry lands before takedown completes, and delisting can take days. The downside risk (total front-of-house outage) is disproportionate to the cost of a domain.
- **PSL entry on `s3.fil.one` without a new domain.** Helps cookie isolation, but many blocklists and filters key on the registrable domain regardless, so `fil.one` remains exposed.
- **A separate domain per regional operator.** Maximum isolation between regions, but fragments the product experience, complicates SDK/endpoint configuration, and multiplies domain operations overhead. Could be a future refinement on top of this proposal.

## Open Questions

- Is `filone.com` the right domain?
- PSL scope: list `filone.com` itself or `s3.filone.com`?

## Evaluation Criteria

- A reputation incident on the data domain (simulated or real) has zero impact on `fil.one` availability, search listing, or email deliverability.
- PSL listing accepted and observed effective in major browsers.
- Console, docs and SDKs emit only new-domain endpoints; old-domain traffic drops to zero by end of the deprecation window.

## References

- [Public Suffix List](https://publicsuffix.org/)
- [Google Safe Browsing](https://safebrowsing.google.com/)
- Precedents: [`amazonaws.com`](https://docs.aws.amazon.com/general/latest/gr/s3.html), [`githubusercontent.com`](https://github.blog/changelog/2021-04-13-user-content-is-served-from-a-new-domain/), `googleusercontent.com`, [`r2.dev`](https://developers.cloudflare.com/r2/buckets/public-buckets/)
