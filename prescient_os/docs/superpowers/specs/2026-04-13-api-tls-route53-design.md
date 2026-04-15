# API TLS And Route53 Design

**Date**: 2026-04-13
**Status**: Approved design for implementation

---

## 1. Goal

Add production-grade DNS and TLS wiring for the AWS API load balancers created by the Terraform app layer.

This follow-on design applies to the existing AWS app roots under:

- `infrastructure/terraform/env/aws/dev/app`
- `infrastructure/terraform/env/aws/staging/app`
- `infrastructure/terraform/env/aws/prod/app`

The goal is to replace the current HTTP-only public ALB exposure with Route53-managed public DNS and ACM-backed HTTPS listeners.

---

## 2. Decisions

### Use Route53 as the authoritative DNS host

The apex domain to manage is:

- `prescientos.ai`

DNS is currently only registered at GoDaddy. Terraform will create the public hosted zone in Route53, but AWS will not become authoritative until the Route53 nameservers are manually configured at GoDaddy.

### Keep the ALB as the public edge for the API

This design does not introduce CloudFront, WAF, or a separate ingress layer. The existing public ALB remains the API entrypoint.

### Terminate TLS on the ALB with ACM

Each environment gets its own ACM certificate in `us-east-1`, validated through Route53 DNS records created by Terraform.

### Scope only the API

This pass wires TLS and DNS for the API hostnames only. It does not add:

- web frontend hosting
- CloudFront
- WAF
- email
- wildcard certificates

Those can be added later without changing the basic hosted-zone or HTTPS-listener model.

---

## 3. Hostname Model

The environment hostnames are:

- `api.dev.prescientos.ai`
- `api.staging.prescientos.ai`
- `api.prescientos.ai`

The apex `prescientos.ai` is not used as the API hostname in this design.

---

## 4. Resource Model

### Route53

Create one public hosted zone for:

- `prescientos.ai`

This hosted zone is shared by all three environments.

### ACM

Each environment app root requests one ACM certificate for its API hostname:

- dev: `api.dev.prescientos.ai`
- staging: `api.staging.prescientos.ai`
- prod: `api.prescientos.ai`

Route53 validation records are created by Terraform in the shared hosted zone.

### ALB

The ALB module is extended to support:

- HTTPS listener on `443`
- ACM certificate attachment
- HTTP listener on `80` that redirects to HTTPS

The ALB security group is extended to allow:

- inbound `80` for redirect traffic
- inbound `443` for real API traffic

Traffic from the ALB to the API target group remains HTTP on port `8000` inside the VPC.

### DNS records

Each environment app root creates an alias record pointing its hostname at that environment’s ALB.

---

## 5. Terraform Ownership

### Shared DNS root

Add a new Terraform root for shared DNS ownership:

- `infrastructure/terraform/env/aws/shared/dns`

This root owns:

- the public Route53 hosted zone for `prescientos.ai`

This root exists because the hosted zone is not environment-specific and should not be duplicated in `dev`, `staging`, or `prod`.

### Per-environment app roots

Each app root owns:

- the ACM certificate for its hostname
- Route53 validation records in the shared zone
- certificate validation resource
- alias record for its API hostname
- ALB HTTPS/redirect configuration

Each app root consumes the shared DNS root via Terraform remote state to obtain:

- hosted zone ID
- hosted zone name

This keeps DNS ownership centralized while letting each environment manage its own hostname and certificate lifecycle.

---

## 6. Remote State And Backend Strategy

The new shared DNS root follows the same remote-state pattern as the other AWS roots:

- explicit S3 backend
- state bucket and lock table matching the existing Terraform account strategy

The approved ownership model is:

- host the Route53 public zone in the `prod` AWS account
- expose a dedicated DNS management role in the `prod` account
- let the `dev`, `staging`, and `prod` app roots assume that role through an aliased AWS provider when they need to create:
  - ACM validation records
  - API alias records

This model was selected because it keeps:

- a single authoritative public zone
- live public DNS for all three environments
- per-environment ownership of certificates and ALBs

without forcing a separate orchestration root to own all records.

Placing the shared DNS root under the `prod` AWS account remains the simplest ownership boundary because:

- public production DNS is typically a production-owned concern
- Route53 hosted zones are global from a DNS perspective

For this implementation pass, the expected outcome is:

- shared hosted zone exists in the `prod` account
- `dev`, `staging`, and `prod` app roots all receive live public DNS and ACM-backed TLS
- cross-account DNS writes happen through an explicit assumable role rather than through duplicated hosted-zone ownership

### Cross-account DNS role

The `prod` account must expose a dedicated role for DNS management. That role should trust the Terraform deployer roles from:

- `dev`
- `staging`
- `prod`

Its permissions should be scoped to Route53 record management for the shared hosted zone. The implementation may keep the first version pragmatic, but the design intent is clear: app roots should not need broad admin access in the DNS-owning account just to write validation and alias records.

---

## 7. Security And Operations

### TLS

Public API traffic must use HTTPS. HTTP should redirect to HTTPS.

### Certificates

ACM handles certificate issuance and renewal automatically after DNS validation succeeds.

### Manual prerequisite

After the hosted zone is created, the Route53 nameservers must be copied into the GoDaddy registrar configuration for `prescientos.ai`. Until that delegation is complete:

- public resolution will not work through Route53
- ACM DNS validation will not complete for public certificates

### Verification

Terraform validation is necessary but not sufficient. A real AWS `plan` or `apply` is required to verify:

- ACM certificate issuance
- Route53 alias creation
- ALB HTTPS listener behavior
- successful HTTP-to-HTTPS redirect

---

## 8. Out Of Scope

This design does not include:

- CloudFront
- WAF
- custom auth at the edge
- frontend/web domain routing
- apex-domain website hosting
- wildcard certificates
- multi-region traffic management

---

## 9. Implementation Summary

Implementation should:

1. Add a shared DNS root for the Route53 hosted zone in the `prod` account.
2. Add the cross-account DNS management role in the `prod` account.
3. Extend the ALB module for HTTPS plus HTTP redirect.
4. Add ACM, Route53 validation, and alias records for all three API hostnames.
5. Expose nameserver outputs so the registrar can be updated at GoDaddy.
