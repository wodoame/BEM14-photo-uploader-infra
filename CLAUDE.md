# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a CloudFormation **infrastructure-only** repository for a containerized photo gallery application deployed on AWS ECS Fargate. Application code lives in a separate repository. Infrastructure is deployed via **CloudFormation GitSync** — the `deployment-file.yaml` at the repo root is the GitSync entry point.

The distinguishing feature of this lab (vs. BEM13) is the use of **nested stacks**: a `root-stack.yaml` references child stacks stored in an S3 bucket, each covering a logical infrastructure layer.

---

## First-time Setup

After cloning, run once to activate git hooks:
```bash
./setup.sh
```

This configures a pre-push hook that automatically syncs changed child stack templates to S3 before every `git push`.

---

## Common Commands

**Lint templates:**
```bash
cfn-lint <template>.yaml
```

**Validate a template:**
```bash
aws cloudformation validate-template --template-body file://<template>.yaml
```

**Package nested stacks** (upload child templates to S3 and rewrite refs):
```bash
aws cloudformation package \
  --template-file root-stack.yaml \
  --s3-bucket <deployment-bucket> \
  --output-template-file packaged.yaml
```

**Deploy manually (for testing, not production):**
```bash
aws cloudformation deploy \
  --template-file packaged.yaml \
  --stack-name bem14-photo-uploader \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides GithubOrg=wodoame GithubAppRepo=<app-repo> Environment=production
```

---

## Nested Stack Architecture

```
root-stack.yaml
├── stacks/network.yaml          # VPC, subnets, IGW, NAT, route tables, VPC endpoints
├── stacks/security.yaml         # All security groups (ALB, ECS, RDS, VPC endpoint SGs)
├── stacks/storage.yaml          # S3 image bucket, CloudFront OAC/OAI, distribution
├── stacks/database.yaml         # RDS PostgreSQL (Multi-AZ, db.t3), subnet group, secret
├── stacks/compute.yaml          # ECR repo, ECS cluster, task definition, ECS service, ALB
├── stacks/autoscaling.yaml      # ECS auto scaling (1 min / 1 desired / 4 max, CPU-based)
└── stacks/cicd.yaml             # CodePipeline, CodeDeploy (blue/green), EventBridge rule
```

**Dependency order** (root-stack passes outputs as parameters to each child):
1. `network` → no dependencies
2. `security` → needs VPC ID from `network`
3. `storage` → no dependencies on network (S3 is global; CloudFront is edge)
4. `database` → needs private subnet IDs and DB security group from `network`/`security`
5. `compute` → needs subnets, security groups, S3 bucket name, CloudFront domain, DB secret ARN
6. `autoscaling` → needs ECS cluster and service name from `compute`
7. `cicd` → needs ECR repo, ECS cluster/service, CodeDeploy app/group from `compute`

---

## Key Design Decisions

- **GitSync entry point**: `deployment-file.yaml` references `root-stack.yaml` as the top-level template; GitSync watches this file to trigger stack updates.
- **Child template storage**: Child stacks must be uploaded to an S3 deployment bucket before GitSync can resolve them. The deployment bucket itself is typically created separately (manually or via a bootstrap stack) before the first GitSync run.
- **S3 image bucket**: Private; accessible only via CloudFront OAC. No public access block is removed — OAC is granted via a bucket policy only.
- **ECS networking**: Tasks in private subnets; ALB in public subnets. VPC endpoints for ECR (API + DKR), S3 (gateway), and CloudWatch Logs keep ECS traffic off the public internet.
- **Blue/green**: CodeDeploy uses `ECS` deployment type. The `appspec.yaml` and `taskdef.json` for deployments are produced by the app CI pipeline (separate repo) and stored as CodePipeline artifacts.
- **OIDC auth**: The app repo's GitHub Actions workflow authenticates to AWS via an IAM OIDC provider — no stored credentials. The OIDC provider and role are defined in `cicd.yaml` or a dedicated `iam.yaml` child stack.
- **CloudFront price class**: `PriceClass_200` (North America, Europe, Asia).
- **RDS**: `db.t3.micro` (or `db.t3.small`), Multi-AZ disabled for cost unless required; single-AZ with a read replica is a valid trade-off.

---

## Repository Layout

```
root-stack.yaml          # Parent stack; references all child stacks via S3 URLs
deployment-file.yaml     # CloudFormation GitSync configuration
stacks/                  # Child stack templates
diagram/                 # Architecture diagram (.drawio)
```
