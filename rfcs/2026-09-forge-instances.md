# RFC: Forge Instances & Pilot Regions

**Status:** Proposal

## Authors

- [Miroslav Bajtoš](https://github.com/bajtos)

## Introduction

We want to operate multiple Forge network instances to serve conflicting needs like continuous
deployment vs stability. This document collects different criteria we have for each instance and
proposes a set of instances to stand up and operate.

The term "Forge network instance" means a collection of one or more regions operating FilOne
appliance, all linked to a single deployment of central components.

## Criteria

### Update frequency

At the moment, we support two options:

1. Fully automated continuous deployment - every commit landed in Forge repositories is deployed in
   minutes.

- Pros: latest & greatest features & bugfixes.
- Cons: less stable with potential outages at unexpected times

2. Scheduled upgrades - at regular intervals, we deploy the latest known-good version.

- Pros: maximum stability, predictable maintenance windows.
- Cons: bugfixes & new features arrive with delay, require regular manual engineering work.

### Stability & acceptable outages

The dev instance provides no guarantees at all.

The production instance must minimise outages.

However, we need also instances on the spectrum between those two options - e.g. an instance running
the latest known-good version where we can perform extensive load testing, an instance running the
latest known-good version which we use for customer demos.

### Infrastructure

Where do the appliances run - Tier 1 providers like AWS, Tier 2 providers, bare-metal?

The dev instance does not need to run on as powerful hardware as we use in production.

On the other hand, performance tests must target an instance that's as close to production infra as
feasible.

### Real vs test money

Forge is integrated with Filecoin Pay, nodes report inclusion proofs to the PDP contract, and the
contract automatically credits node operators for utilised storage space.

Each Forge instance is tied to one Filecoin chain (mainnet or calibration). Instances tied to the
mainnet must deal with real funds - periodically top up the wallets paying for storage and for gas,
using real FIL.

On the FilOne side, we use Stripe sandbox in non-production environments, which gives use "test"
money and test credit card numbers we can use to pay for storage. This makes it easy to test FilOne
& Forge for free, with no real credit card needed.

### Data retention

Non-production instances should have limited data retention period and we should implement regular
resets. This is needed to keep the used storage reasonable and prevent abuse.

However, data resets will break demos and long-term performance tests, therefore the schedule must
be based on other criteria like stability.

### Regions

How many regions and in which geographic location?

## Proposal

Provision the following three Forge instances:

- production
- dev
- staging

Additionally, to support customer pilots, implement support for short-lived pilot regions available
to selected customers in the production Forge instance.

### 1. Production

This is the instance our paying customers use.

- **Update frequency:** Manually triggered updates. Schedule will be determined later (see open
  questions below).
- **Stability & acceptable outages:** Maximum stability, maintenance windows outside of core
  business hours, minimum downtime. Full monitoring with alerts routed to the person on pager duty.
- **Infrastructure:** The central components running on AWS ECS must have enough capacity to handle
  the entire production workload. Regional appliances run in node operators' datacenters and must
  have enough power to administer their storage capacity.
- **FilOne integration**: Available via the production console at https://app.fil.one. Available to
  all users.
- **Real vs test money:** Real money, Filecoin mainnet.
- **Data resets:** none
- **RPC API:** Lotus node running in each region.

Note: we need to define the process for shipping hotfixes outside of the regular update schedule.
The longer the interval between regular updates, the higher the chance that we need a hotfix.

#### Pilot Regions

We need the ability to quickly stand up new regions to allow potential customers evaluate FilOne in
pilot/proof-of-concept settings. These regions must be production-grade deployment matching the real
production nodes as much as possible.

Typically, we will need to quickly stand up a new pilot region within 72 hours, keep it running &
meeting SLAs for 90 days, and decommission it after that.

We will _not_ offer data migration from pilot regions to production.

- **Update frequency:** Manually triggered updates. Typically once before the pilot starts and then
  when the customer requests new features or we need to fix bugs.
- **Stability & acceptable outages:** Same guarantees as in production. Full monitoring with alerts routed to the person on pager duty.
- **Infrastructure:** Production-like. Nodes will run in Tier-2 infra providers like Vulture, Akamai
  or servers.com.
- **FilOne integration**: Available via the production console at https://app.fil.one, with region
  names suffixed with `-pilot`, e.g. `uk-1-pilot`. Available only to selected users via a feature
  flag.
- **Real vs test money:** Real money, Filecoin mainnet. We can apply a discount coupon on Stripe to
  give the customer a free pilot.
- **Data resets:** None during the pilot duration. Data will be removed after the pilot has
  finished.
- **Regions:** Created on demand.
- **RPC API:** TBD. Ideally, each region should run a local Lotus node instance. Can we afford the
  cost and maintenance overhead of that?

Important: by adding pilot regions to the production Forge instance, we allow customers to create S3
access keys with access to both dev & staging regions.

### 2. Dev

This is the instance where we continuously ship all changes.

- **Update frequency:** Every change is deployed as soon as feasible.
- **Stability & acceptable outages:** No stability guarantees.
- **Infrastructure:** The Appliance is running on a relatively small AWS EC2 instance. We support
  light testing, but not performance/load testing. Light monitoring if any at all.
- **FilOne integration**: Available via the staging console at https://staging.fil.one as the region
  `us-east-9`. Available to all users.
- **Real vs test money:** Stripe test cards, Filecoin calibnet.
- **Data retention:** Weekly network reset on Sunday morning UTC.
- **Regions:** Single region (`us-east-9`).
- **RPC API:** Chain.Love.

### 3. Staging

This is a stable "preview" instance showing the latest & greatest features, suitable for customer
demos. Not used for load/performance testing to avoid degraded performance during demos.

- **Update frequency:** Manually triggered updates. Schedule will be determined later (see open
  questions below).
- **Stability & acceptable outages:** Reasonable stability and minimum unplanned downtime. Full
  monitoring with alerts routed to the person on pager duty, with capped severity (no incident is
  critical).
- **Infrastructure:** The servers.com baremetal box in Amsterdam where Filecoin Foundation operates
  a Calibnet SP.
- **FilOne integration**: Available via the staging console at https://staging.fil.one as the region
  `eu-central-3`. Available to all users.
- **Real vs test money:** Stripe test cards, Filecoin calibnet.
- **Data resets:** TBD. Monthly resets?
- **Regions:** eu-central-3, potentially more in the future.
- **RPC API:** Local Lotus node in `eu-central-3`. To be determined for future regions.

Important: S3 access keys are scoped to a single Forge instance. It won't be possible to create one
S3 access key with access to both dev & staging regions.

## Open questions

- Update frequency for production and staging environments. Do we upgrade regularly (e.g. every
  week, biweekly on sprint end, monthly), more often or less frequently?
- How often we reset data & state in the staging environment.
- How many Lotus nodes we want to run (per-region vs one central instance)? Where can we outsource
  this to Chain.Love/Protofire?

## Alternatives considered

### Dedicated Forge instance for pilot regions

Pros: Pilots are isolated from production.

- A run-away load test in pilot region does not affect production customers.
- We can ship changes to central services without risks of breaking production.

Cons:

- Worse user experience, including feature limitations:
  - A single S3 access key cannot be scoped to access buckets in both production and pilot regions.
  - Different S3 endpoint hostname format production vs pilot regions.
- Maintenance overhead - we need to maintain & monitor another set of central services.
