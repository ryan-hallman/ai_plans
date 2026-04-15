# Terraform Infrastructure Design

**Date**: 2026-04-13
**Status**: Approved design — pending implementation planning

---

## 1. Goal

Add an infrastructure-as-code foundation for Prescient OS that is:

- **Local-first today** — developers can provision AWS-shaped local infrastructure against LocalStack without waiting for cloud deployment work
- **Enterprise-safe later** — real AWS environments use separate accounts, separate state, GitHub Actions-driven deployment, and clear stack boundaries
- **Low-refactor by design** — the initial repository layout and Terraform structure should scale into multi-account AWS without requiring a later reorganization of roots, modules, or CI/CD

This design assumes:

- Local development remains Docker-first for the application runtime
- Real AWS deployment will use **one account per environment**
- The cloud runtime will be **ECS/Fargate** for services and **SQS** for async work
- **EventBridge** is used for scheduling and event routing, not as the durable work queue
- GitHub Actions is the only path for applying Terraform to real AWS environments

---

## 2. Design Decisions

### Use plain Terraform, not Terragrunt

The repository is not yet large enough to justify Terragrunt's extra abstraction. A single repo with explicit environment roots and reusable modules is sufficient for:

- `local`
- `dev`
- `staging`
- `prod`

Terragrunt remains a possible later optimization, but it is not required for a correct day-1 multi-account design.

### Keep infrastructure in this repository

Terraform lives beside the application code rather than in a separate infra repo. This reduces context switching while the platform is still evolving rapidly. The layout is intentionally structured so it can be split into a dedicated infra repo later if ownership or deployment cadence diverges.

### Use explicit root stacks, not Terraform workspaces

Long-lived environments are modeled as separate root directories with separate state. Terraform workspaces are not the primary environment isolation model.

### Treat local as a first-class Terraform target

`local` is a real Terraform root that targets LocalStack where practical. It is developer-operated and exposed through `Makefile` targets rather than GitHub Actions.

---

## 3. Repository Layout

Terraform lives under:

```text
infrastructure/terraform/
├── modules/
└── env/
```

### Reusable modules

```text
infrastructure/terraform/modules/
├── network-base/
├── postgres/
├── redis/
├── opensearch/
├── object_store/
├── ecs_service/
├── ecs_worker/
├── sqs_queue/
├── eventbridge_schedule/
├── alb_service/
└── app_secrets/
```

Module names may change slightly during implementation, but the boundary rule is fixed: modules are reusable building blocks, not environment roots.

### Deployable roots

```text
infrastructure/terraform/env/
├── local/
└── aws/
    ├── dev/
    │   ├── bootstrap/
    │   ├── network/
    │   ├── data/
    │   └── app/
    ├── staging/
    │   ├── bootstrap/
    │   ├── network/
    │   ├── data/
    │   └── app/
    └── prod/
        ├── bootstrap/
        ├── network/
        ├── data/
        └── app/
```

Each root is independently deployable and owns a single responsibility boundary.

---

## 4. Environment Model

### AWS environments

Each real environment maps to a separate AWS account:

- `dev`
- `staging`
- `prod`

This is the primary isolation boundary for permissions, spend, blast radius, and secrets.

### Local environment

`local` is not modeled as an AWS account. It is a development target designed for:

- LocalStack-backed AWS integrations where practical
- Docker Compose-backed local runtime for Postgres, Redis, and OpenSearch
- Fast feedback for engineers without involving cloud accounts or CI

---

## 5. State Strategy

Each AWS root stack gets its own Terraform state. State is separated by both environment and layer:

- `dev/bootstrap`
- `dev/network`
- `dev/data`
- `dev/app`
- `staging/bootstrap`
- `prod/app`
- etc.

This keeps state locking scoped, reduces blast radius, and allows independent rollouts by layer.

### AWS state backend

For real AWS environments:

- Use **S3** for Terraform state
- Use **DynamoDB** for state locking

### Bootstrap caveat

Terraform cannot use an S3 backend that does not exist yet. The bootstrap flow must account for that. The expected approach is:

1. Intentionally establish backend primitives per account
2. Migrate non-bootstrap stacks to the remote backend
3. Keep subsequent applies fully GitHub Actions-driven

The exact bootstrap mechanics can be finalized in the implementation plan, but the design requirement is clear: backend setup must be deliberate and must not rely on ad hoc engineer-specific workflows.

### Local state

For `env/local`:

- Use local Terraform state files
- Do not use remote state
- Keep the environment disposable and developer-oriented

---

## 6. Stack Layers

Each AWS environment contains four root stack layers.

### `bootstrap`

Owns account-level and deployment baseline concerns:

- GitHub Actions OIDC deploy role
- Baseline IAM roles and policies required for deploys
- KMS keys if customer-managed encryption is enabled from the start
- ECR repositories
- Backend support primitives where appropriate

### `network`

Owns shared network topology:

- VPC
- public and private subnets
- route tables
- NAT strategy
- security groups
- VPC endpoints as needed

### `data`

Owns durable stores and configuration services:

- RDS PostgreSQL
- ElastiCache Redis
- OpenSearch
- S3 buckets
- Secrets Manager and/or SSM Parameter Store
- subnet groups and security groups related to data services

### `app`

Owns runtime services and async orchestration:

- ECS cluster
- Fargate services for `api` and `web`
- worker services for asynchronous processing
- SQS queues and DLQs
- EventBridge schedules and rules
- ALB and target groups
- task definitions
- autoscaling configuration
- task and service IAM roles
- runtime wiring from `network` and `data` outputs

### Why only four layers

The current repository shape does not justify splitting `search`, `async`, `edge`, or `observability` into their own roots yet. The app is clearly multi-service, but not so large that additional root layers improve clarity more than they add coordination overhead.

---

## 7. Service Mapping From The Existing Codebase

This design is grounded in the current repository and phase plans:

- **Postgres** is the source of truth across bounded contexts and becomes **RDS**
- **Redis** already exists in local runtime and becomes **ElastiCache**
- **OpenSearch** is already load-bearing for document indexing and recall
- **S3** is introduced now because it supports artifact/file storage, exports, caches, and future upload flows without later shape changes
- **ECS/Fargate** fits the existing Dockerized `api` and `web` services better than Lambda or Kubernetes
- **SQS** supports the emerging async workload pattern: research jobs, indexing, refreshes, briefing assembly, and outbox-driven processing
- **EventBridge** supports schedules and event routing, including monitoring cadence and daily briefing generation

This means the day-1 AWS target service set is:

- VPC
- ECS/Fargate
- ECR
- RDS
- ElastiCache
- OpenSearch
- S3
- SQS
- EventBridge
- IAM
- KMS
- Secrets Manager / SSM
- ALB
- CloudWatch

Later services such as Route53, ACM, CloudFront, and SES can be added without changing the root stack model.

---

## 8. LocalStack Strategy

The rule for local development is:

> Everything that can reasonably target LocalStack should do so. Everything that is materially simpler and more reliable in Docker today should remain in Docker.

### LocalStack-backed in `env/local`

Use LocalStack for AWS-native services where it provides meaningful development value:

- S3
- SQS
- EventBridge
- Secrets Manager
- IAM-adjacent wiring as needed for Terraform composition

ECS support may be explored for parity, but it is not required for the first implementation if it increases local complexity without improving developer workflows.

### Docker Compose-backed in local runtime

Keep these in Docker Compose for local development:

- Postgres
- Redis
- OpenSearch
- API container
- Web container

This matches the current repository's local runtime and avoids turning local development into a brittle full-cloud emulation project.

### Local Terraform responsibility

`env/local` provisions local AWS-shaped resources and any related configuration needed for developers. It does not replace Docker Compose as the runtime orchestrator for the app and core data stores.

---

## 9. CI/CD And Deployment Model

### AWS applies go through GitHub Actions

Terraform applies to real AWS accounts must be executed by GitHub Actions, not by engineers from local machines.

Engineers may still run local validation workflows such as:

- `terraform fmt`
- `terraform validate`
- `terraform plan`

But real AWS `apply` flows are centralized in CI/CD.

### Authentication model

GitHub Actions assumes a role in each AWS account using OIDC:

- no long-lived AWS credentials in GitHub secrets
- one deploy role per target account
- production deploys protected by GitHub environment approvals

### Stack ordering

Deployment order is explicit:

1. `bootstrap`
2. `network`
3. `data`
4. `app`

### Environment promotion

Recommended rollout behavior:

- PRs touching Terraform run formatting, validation, and scoped plans
- merge to `main` can auto-apply `dev`
- `staging` and `prod` require approval gates

### Container delivery

GitHub Actions builds and pushes API and web images to ECR. The `app` stack consumes explicit image references rather than building containers at apply time.

---

## 10. Environment Differences

Environment-specific variation should be expressed primarily as data, not structure.

Examples of allowed per-environment differences:

- instance and task sizing
- retention settings
- scaling thresholds
- domain names
- bucket names
- alarm thresholds
- feature flags

Examples of differences to avoid:

- different root stack shapes per environment
- hand-maintained drift in module composition
- environment-specific logic forks unless the capability truly does not exist in that environment

The default posture is:

- same module graph across `dev`, `staging`, and `prod`
- different values where appropriate

---

## 11. Makefile Integration For Local Terraform

Local Terraform workflows are exposed through the repository `Makefile`, not GitHub Actions.

Planned targets:

- `make tf-local-init`
- `make tf-local-plan`
- `make tf-local-apply`
- `make tf-local-destroy`

These targets should wrap the `env/local` Terraform root and give developers a predictable entry point that matches the rest of the repository's Docker-first ergonomics.

---

## 12. Non-Goals

This design intentionally does **not** do the following:

- introduce Terragrunt on day 1
- create a separate infrastructure repository on day 1
- use Terraform workspaces as the primary environment model
- force full LocalStack parity for services already served better by Docker in local development
- let engineers apply real AWS infrastructure directly from laptops as the standard workflow

---

## 13. Growth Path

This design is intentionally shaped so future changes are additive rather than structural:

- Terragrunt can be added later if stack count and repetition justify it
- `search`, `async`, `edge`, or `observability` can split into separate roots if the platform grows
- a dedicated infra repo can be introduced later because reusable modules and environment roots are already isolated
- additional AWS environments or accounts can be added by repeating the existing `env/aws/<environment>/<layer>` pattern

The main goal is to avoid a future “big Terraform rewrite” by choosing a root/module/state model that already fits the expected cloud operating model.
