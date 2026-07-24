# Three-Tier AWS Architecture — Terraform

Terraform (JSON syntax) that provisions a three-tier web application architecture on AWS: public ALB tier, private application tier (Auto Scaling Group), and a private data tier (RDS Multi-AZ), fronted by CloudFront + WAF and exposed via Route 53.

## Architecture

- **Networking** — VPC with 2 public, 2 app (private), and 2 data (private) subnets across 2 AZs, an Internet Gateway, and one NAT Gateway per AZ.
- **Security Groups** — ALB (80/443 from internet) → App tier (80 from ALB SG only) → DB tier (3306 from App SG only).
- **Application tier** — Application Load Balancer + Launch Template (Amazon Linux 2023) + Auto Scaling Group with target-tracking scaling on CPU (60%).
- **Data tier** — RDS MySQL 8.0, Multi-AZ, encrypted storage, deployed in the private data subnets.
- **Edge / Security** — AWS WAFv2 (managed rule sets + rate limiting) attached to a CloudFront distribution in front of the ALB.
- **DNS** — Route 53 alias record pointing the domain at CloudFront.
- **Observability** — CloudWatch log group, alarms (ASG CPU, RDS CPU, ALB 5xx), and a dashboard.

## File layout

| File | Purpose |
|---|---|
| `00_provider.tf.json` | Terraform/provider config, AMI + AZ data sources |
| `01_variables.tf.json` | Input variables |
| `02_vpc.tf.json` | VPC, subnets, IGW, NAT, route tables |
| `03_security_groups.tf.json` | ALB / App / DB security groups |
| `04_alb.tf.json` | Application Load Balancer, target group, listener |
| `05_asg.tf.json` | Launch template, Auto Scaling Group, scaling policy |
| `06_rds.tf.json` | RDS subnet group + MySQL instance |
| `07_waf.tf.json` | WAFv2 Web ACL (CloudFront scope) |
| `08_cloudfront.tf.json` | CloudFront distribution |
| `09_route53.tf.json` | Route 53 alias record |
| `10_cloudwatch.tf.json` | Log group, alarms, dashboard |
| `11_outputs.tf.json` | Terraform outputs |

## Prerequisites

- Terraform >= 1.5.0
- An AWS account/credentials with permission to create the above resources
- A domain already hosted in Route 53 (public hosted zone)

## Required variables

Most variables have sensible defaults (see `01_variables.tf.json`). Two have **no default** and must be supplied:

| Variable | Description |
|---|---|
| `db_password` | RDS master password (sensitive) |
| `domain_name` | Root domain in Route 53, e.g. `example.com` |

Provide them via `terraform.tfvars` (kept out of git) or environment variables:

```bash
export TF_VAR_db_password="your-strong-password"
export TF_VAR_domain_name="example.com"
```

## Usage

```bash
terraform init
terraform validate
terraform plan
terraform apply
```

To tear down:

```bash
terraform destroy
```

## CI/CD (Jenkins)

A `Jenkinsfile` is included and runs: checkout → `fmt -check` → `init` → `validate` → `plan` → manual approval → `apply` or `destroy`.

**Jenkins credentials required** (Manage Jenkins → Credentials):

| Credential ID | Type | Value |
|---|---|---|
| `aws-access-key-id` | Secret text | AWS access key |
| `aws-secret-access-key` | Secret text | AWS secret key |
| `tf-db-password` | Secret text | RDS master password |
| `tf-domain-name` | Secret text | Route 53 domain name |

**Pipeline parameters:**
- `ACTION` — `plan`, `apply`, or `destroy` (default `plan`)
- `AUTO_APPROVE` — skip the manual approval gate (default `false`)

Set up the job as a Pipeline pointing at this repo's `Jenkinsfile`, or as a Multibranch Pipeline.

## Notes / things to review before production use

- `skip_final_snapshot = true` and `deletion_protection = false` on RDS — change these before using in production.
- The ALB listener is HTTP-only (port 80); add an HTTPS listener + ACM certificate for TLS to the ALB, and consider requiring HTTPS end-to-end.
- CloudFront's `viewer_certificate` uses the default CloudFront certificate — attach an ACM certificate in `us-east-1` for the custom domain to serve HTTPS correctly.
- State is not currently configured with a remote backend (S3 + DynamoDB lock table recommended) — add one before running this from CI.
