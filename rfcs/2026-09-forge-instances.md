# RFC: Forge Instances

**Status:** Proposal

## Authors

- [Miroslav Bajtoš](https://github.com/bajtos)

## Introduction

We want to operate multiple Forge network instances to serve conflicting needs like continuous
deployment vs stability. This document collects different criteria we have for each instance and
proposes a set of instances to stand up and operate.

The term "Forge network instance" means a collection one or more regions operating FilOne appliance,
all linked to a single deployment of central components.

## Criteria

### Update frequency

At the moment, we support two options:

1. Fully automated continuous deployment - every commit landed in Forge repositories is deployed in minutes.

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

On the other hand, performance tests must target an instance that's as close to production infra as feasible.

### Real vs test money

Forge is integrated with Filecoin Pay, nodes report inclusion proofs to the PDP contract, and the
contract automatically credits node operators for utilised storage space.

Each Forge instance is tied to one Filecoin chain (mainnet or calibration). Instances tied to the
mainnet must deal with real funds - periodically top up the wallets paying for storage and for gas,
using real FIL.

On the FilOne side, we use Stripe sandbox in non-production environments, which gives use "testest"
money and test credit card numbers we can use to pay for storage. This makes it easy to test FilOne
& Forge for free, with no real credit card needed.

### Data resets

Non-production instances should have limited data retention period and we should implement regular
resets. This is needed to keep the used storage reasonable and prevent abuse.

However, data resets will break demos and long-term performance tests, therefore the schedule must
be based on other criteria like stability.

### Regions

How many regions and in which geographic location?

### 1. Production

This is the instance our paying customers use.

- **Update frequency:** TBD. Once per month? When is the scheduled maintenance window for the central and individual regions?
- **Stability & acceptable outages:** Maximum stability, maintenance windows outside of core
  business hours, minimum downtime. Full monitoring with alerts routed to the person on the pager duty.
- **Infrastructure:** The central components running on AWS ECS must have enough capacity to handle the
  entire production workload. Regional appliances run in node operators' datacenters and must have
  enough power to administer their storage capacity.
- **Real vs test money:** Real money, Filecoin mainnet.
- **Data resets:** none

Accessible from FilOne at https://app.fil.one (the main app).

Note: we need to define the process for shipping hotfixes outside of the regular update schedule.
The longer the interval between regular updates, the higher the chance that we need a hotfix.

### 2. Dev

This is the instance where we continuously ship all changes.

- **Update frequency:** Every change is deployed as soon as feasible.
- **Stability & acceptable outages:** No stability guarantees.
- **Infrastructure:** The Appliance is running on a relatively small AWS EC2 instance. We support
  light testing, but not performance/load testing. Light monitoring if any at all.
- **Real vs test money:** Stripe test cards, Filecoin calibnet.
- **Data rention:** Weekly network reset on Sunday morning UTC.
- **Regions:** Single region (us-east-9).

Accessible from all non-production FilOne deployments (e.g. https://staging.fil.one, but also PR
previews) as the region `us-east-9`.

### 3. Staging

This is a stable "preview" instance showing the latest & greatest features, suitable for customer
demos. Not used for load/performance testing to avoid degraded performance during demos.

- **Update frequency:** Every Monday morning UTC. Can be rescheduled ad-hoc in case of a customer demo planned for Monday.
- **Stability & acceptable outages:** Reasonable stability and minimum unplanned downtime. Full monitoring with alerts routed to the person on the pager duty, with capped severity (no incident is critical).
- **Infrastructure:** TBD.
- **Real vs test money:** Stripe test cards, Filecoin calibnet.
- **Data resets:** TBD. Monthly resets?
- **Regions:** TBD (Right now, we can have eu-central-3 on a powerful servers.com bare-metal box.)

### 4. Performance testing

This is the instance we will use for intensive load & performance testing.

- **Update frequency:** Manual, before starting a new test run.
- **Stability & acceptable outages:** No stability guarantees besides minimising unplanned downtime. Full monitoring but without alerts.
- **Infrastructure:** Production-like. Can we run a perf-test appliance in each production datacenter?
- **Real vs test money:** Stripe test cards, Filecoin calibnet.
- **Data resets:** Manual, after finishing a test run.
- **Regions:** Ideally the same regions as in production.
