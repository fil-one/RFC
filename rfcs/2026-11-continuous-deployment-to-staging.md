# RFC: Continuous Deployment to Staging

**Status:** Proposal

## Authors

- [Miroslav Bajtoš](https://github.com/bajtos)

## Introduction

We operate three Forge network instances: production, staging and dev. Every change merged in a
Forge repository is deployed to dev within minutes. A bot then opens a pull request that promotes
the change to staging, and a person merges it.

This RFC proposes to remove the dev instance and deploy every change to staging automatically. It
supersedes parts of [Forge Instances & Pilot Regions](./2026-09-forge-instances.md) and of other
earlier RFCs, listed in [Superseded parts of earlier RFCs](#superseded-parts-of-earlier-rfcs).

## Motivation

1. **Promotions to staging wait for a person.** Somebody has to remember to merge the promotion
   pull requests. When nobody does, staging falls behind dev.
2. **A network reset is done twice.** When an upgrade of the network configuration needs a reset,
   we reset dev and then repeat the work on staging.
3. **Nothing outside the team uses either network.** Forge Instances defined staging as a stable
   preview for customer demos. We never used it that way. With no external user of dev or staging,
   we have no need for a semi-stable instance next to an unstable one.
4. **Every instance costs upkeep.** Dev is a third set of central services and a third appliance to
   pay for, upgrade, monitor and document.

## Goals

- A change merged to `main` in a Forge service repository or infrastructure repository runs on
  staging within minutes, with no action from a person.
- We operate two long-lived Forge instances: staging and production.
- One manual step stands between a merged change and production.
- Staging is the place for destructive work: network resets, destructive tests and rehearsals of
  risky upgrades.

### Out of scope

- How and when production is updated.
- Pilot instances.
- Where customer demos run. See [Open questions](#open-questions).

## Proposal

### Remove the dev instance

Shut down the dev central services and the dev appliance, and remove everything that refers to
them. The region `us-east-9` leaves the staging console. Its tenants, buckets and objects are
discarded with no migration to staging.

The staging console at https://staging.fil.one, and every other non-production deployment of the
console, then offers one Forge network with the region `eu-central-3`.

### Deploy every change to staging

The automation that deploys to dev today targets staging:

- A new image published from a service's `main` branch opens a pull request that pins the image on
  staging. The pull request merges on its own once CI passes.
- A change to the infrastructure repositories is applied to staging when it merges.
- The tests run against staging after each deploy. When they pass, a bot opens the pull request
  that promotes the same versions to production.

A person merges the production promotion. That merge is the only manual step.

### Staging is the lowest environment

Staging takes over what dev was for, and gives up the role Forge Instances gave it.

- **No stability guarantees.** Staging runs whatever was merged last. It is no longer a stable
  preview and it is no longer updated by hand.
- **Resets on demand.** We reset staging when a change needs it. There is no reset schedule.
- **Destructive tests run against it.** The tests that lock and disable tenants keep running, now
  against staging only.
- **Staging can be rebuilt.** We remove the protections that stop us from rebuilding staging's
  central database and platform, and we allow major-version upgrades to be applied there directly.
  A network reset needs this, staging is now the only hosted place to rehearse a risky upgrade, and
  staging holds test data only.
- **Backups stay.** Staging keeps taking backups, because the restore drills described in
  [Appliance Layout and Disaster Recovery](./2026-10-appliance-layout-and-disaster-recovery.md) run
  there.
- **Alerting stays as it is.** Staging failures keep alerting the team. A broken build on `main`
  now raises a staging alert. Dev alerted on nothing, so the same build used to fail unnoticed.

### Remove personal sandbox stages

We documented a way to stand up a personal copy of the central services under the `dev` DNS label.
Nobody has used it so far, and a local [smelt](https://github.com/fil-forge/smelt) network already serves the
purpose of trying a change before it merges. The procedure and the `dev` DNS zones are removed
together with the dev instance.

## Trade-offs

- **A change reaches staging without running on any other hosted network first.** The Fil One
  console's production deploy waits for end-to-end tests on https://staging.fil.one, and those tests
  use the Forge network behind it. A broken Forge build can therefore hold a console release until
  it is fixed or reverted. The same is true today: the tests also cover `us-east-9`, which runs
  every build.
- **We have no stable non-production instance.** Anything that needs one, such as a customer demo
  or a long-running performance test, has no home until we decide where it goes.

## Superseded parts of earlier RFCs

This RFC supersedes the following:

- [Forge Instances & Pilot Regions](./2026-09-forge-instances.md): the dev instance (section 2);
  the update frequency, stability and data resets of staging (section 3); and the note about access
  keys scoped to dev or staging.
- [Forge-Central Deployment Strategy](./2026-08-forge-central-deployment-strategy.md): continuous
  delivery to the dev environment, and promotion from dev to staging and production.
- [Fil One Appliance Deployment Strategy](./2026-08-fil-one-appliance-deployment-strategy.md): the
  dev node, continuous deployment to it, promotion of versions tested in dev, and upgrade
  rehearsals in dev.
- [Appliance Layout and Disaster Recovery](./2026-10-appliance-layout-and-disaster-recovery.md):
  the dev layout on AWS EC2.
- [Forge Network Service Identities](./2026-07-forge-service-identities.md): the `dev` label for
  ephemeral and personal stages.

## Alternatives considered

### Keep dev and merge the staging promotion automatically

The smallest change: leave the three instances in place and let the bot merge the dev-to-staging
promotion once dev passes its tests.

Rejected because it removes only the forgotten merges. We would still reset two networks, still
maintain a third instance, and still keep a semi-stable tier nobody uses.

### Keep personal sandbox stages

Keep the procedure and the `dev` DNS zones, and base a personal stage on staging's configuration.

Rejected because nobody has used a personal stage so far and smelt covers the need.

### Keep staging's protections and rehearse elsewhere

Leave staging's database protected against deletion and rehearse risky upgrades in a personal
stage.

Rejected because staging has to be resettable, it holds nothing that needs protecting, and it is
the one hosted instance left to rehearse on.

### Remove staging's backups too

Drop the backup retention lock and the replicas so that staging can be destroyed completely.

Rejected because the restore drills need a hosted instance that takes real backups.

### Test a build before it reaches staging

Add a gate in front of the automatic merge, such as an end-to-end run on a smelt network.

Rejected for now. We accept that a broken build reaches staging and alerts the team.

### Wait before promoting to production

Open the production promotion only after a build has run on staging for a set time.

Rejected. The person who merges the promotion decides whether a build has run long enough.

### Reset staging on a schedule

Forge Instances gave dev a weekly reset.

Rejected for now. The weekly reset was never implemented for dev, because the need for it did not
arise. We can add a schedule to staging later, when we need one to reduce the storage used or to
deal with abuse.

## Open questions

- Where do customer demos run? Staging gives up that role. We will answer this when there is demand
  for demos.
