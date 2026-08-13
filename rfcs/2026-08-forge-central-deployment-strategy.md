# RFC: Forge-Central Deployment Strategy

**Status:** Proposal

## Authors

- [Miroslav Bajtoš](https://github.com/bajtos)

## Motivation

Forge consists of two independent sets of services: Forge-Central contains centrally managed set of
services operated by the FilOne team, FilOne Appliance contains the software stack powering regional
nodes.

This pull request proposes deployment strategy for the Central stack. It's a complement for
[rfc#19](https://github.com/fil-one/RFC/pull/19), which describes the deployment strategy for FilOne
Appliance.

## Requirements & Constraints

1. All aspects of the deployment are managed using infrastructure-as-code, version-tracked in Git.

1. Continuous delivery for the dev environment. Every commit landed on `main` in any of the service
   repositories is eventually deployed to the dev env with no manual steps involved.

1. We can promote a given combination of service versions & configurations from dev env to other
   environments (staging, production) as a single git commit.

### Non-requirements

The following requirements are out of the scope of the initial version:

1. Zero-downtime upgrades
2. High availability / no single points of failures.
3. Multi-region deployment.
4. Local Lotus node providing Filecoin RPC

## Components & Dependencies

Forge Central contains the following services:

| Service              | Dependencies                    | Interacts with    |
| -------------------- | ------------------------------- | ----------------- |
| sprue                | Postgres, 3 S3 buckets          | plc               |
| hilt                 | Postgres, OpenBao               | sprue, plc, swarf |
| swarf                | Postgres                        | plc               |
| delegator            | 2 DynamoDB tables, Filecoin RPC |                   |
| piri-signing-service | Filecoin RPC                    |                   |
| plc                  | Postgres                        |                   |

## Proposal

Adopt a cloud-native architecture that allows us to push a lot of the operational burden to AWS
while preventing a vendor lock-in that would make it difficult to migrate to a different cloud
provider or onprem.

- Run all Forge services as idenpendent OCI containers on AWS ECS Fargate
- A single Postgres on AWS RDS, shared by all services, with one database per service for isolation
- OpenBao running on AWS ECS Fargate, unsealed automatically using AWS KMS
- AWS S3 for sprue
- AWS DynamoDB for delegator
- Manage all resources using Terraform

### High-level Layout

**Terraform Workspaces**

Split the components into three Terraform workspaces:

1. Bootstrap: ECR repository per AWS Account & Region.

2. Platform: 3rd-party dependencies

   - Postgres
   - OpenBao
   - AWS KMS
   - AWS DynamoDB & S3
   - AWS ALB
   - AWS VPC & NAT

3. Apps: Forge services
   - sprue
   - hilt
   - swarf
   - delegator
   - signing-service
   - plc

This allows us to scope changes and limit their blast radius - a mistake in apps configuration will not accidentally drop or recreate the DB server.

**Terraform Modules**

Define the infrastructure resources in modules shared by all environments.

Each environment is described by a new set of Terraform files that define only the parameters like
multi-AZ database, deletion protection and different VM sizing.

### Secret & Key Generation

When provisioning a new environment, we need to generate several secrets - wallet private keys and
identity private keys. These secrets must not leak and must not be stored anywhere outside of AWS
SSM or OpenBao, in particular they must not be written to local filesystem or make it to the
Terraform state.

We will achieve this by implementing a `provision` Lambda function (written in Go) that will
generate all secrets and write them to AWS SSM. The Terraform will invoke this Lambda function when
provisioning a new deployment.

### Continuous Delivery

Terraform files for Forge Central will live in a dedicated GitHub repository - https://github.com/fil-forge/infra-central.

Every commit landed to the `main` branch in this repository will trigge `terraform plan` and
`terraform apply` for the dev workspaces (platform, apps).

We will create GitHub Actions-based pipeline to automatically promote changes from individual
service repositories: when a new image version is pushed to GHCR in a service repository, a pull
request will be automatically opened in the infra-central repository to bump the pinned digest. This
pull request will be automatically merged after it passes CI checks.

## Alternatives Considered

**[SST](https://sst.dev/)**

Rejected: SST's main focus is on serverless projects written in TypeScript.

**[Storoku](https://github.com/storacha/storoku)**

Rejected:

- Feature gaps:
  - Storoku models one public HTTP service per repository, with its own VPC, ECS cluster, ALB, and so on. Our project is seven services sharing one stage.
  - We need to fetch secrets from SSM and write them to config files when services start. Storoku has no entryPoint/command override; nothing writes secrets to files.
  - Storoku hardcodes the healtcheck enpoint, while Forge services use multiple different paths
  - No general-purpose Lambda we could use to provision secrets
  - Autoscaling with blue/green deployments is not compatible with our DB migrations that require `desired_count = 1`
- Storoku adds another layer of abstraction that both humans and coding agents would need to learn.
- Coding agents can handle the verbosity of Terraform for us, there is less need for prefabricated resource bundles.
- Storoku is not maintaned. The only bugfixes and improvements we will get are those we make ourselves.

However, we will review the best practices and security hardening that went into Storoku and apply
it to our Terraform projects.

**[Pulumi](https://www.pulumi.com/)**

Rejected? Discussing this on Slack.

Pulumi has two main advantages:

- We can use our language of choice (Go) instead of HCL, which makes it easier to create reusable components.
- We can write unit test for our infrastructure definitions and run them as part of regular "go test" workflow.

It also comes with downsides:

- We are already using Terraform to manage FilOne Cloudflare domains; we will end up having both Terraform and Pulumi in our toolbelt.

- Pulumi is primarily TypeScript. Some plugins like awsx require Node.js runtime even if we describe
  our Pulumi resources in Go. Either we will be limited to core features only, or we have to add
  Node.js to our dependencies. (Adding Node.js is not a big deal to me, but it's a bit ugly.)

- If we want to use Pulumi Cloud, it's yet another service to procure, and they charge per resource
  managed (not per-seat/per-run as Hashicorp), which get expensive as the stack grows. (I am not
  sure if this is applicable to us, though.) The alternative is DIY backend on S3 + KMS. We will
  lose RBAC beyond S3 bucket IAM, no web UI, no deployment history view.

- Pulumi's AWS provider is largely bridged from the Terraform provider, so coverage is equivalent,
  but bug fixes occasionally arrive with a lag, and error messages sometimes leak Terraform-isms.
