# Infrastructure (Terraform)

## Overview
Terraform provisions the AWS infrastructure needed to run the application using an **EC2 Auto Scaling Group (ASG)** behind an **Application Load Balancer (ALB)**. Container images are stored in **Amazon ECR**. An optional **RDS PostgreSQL** database can be enabled.

Region: **eu-north-1**

---

## Components created
- VPC, subnets, routing (module)
- Security groups (ALB + EC2 + optional RDS)
- Application Load Balancer (ALB)
- Launch Template + Auto Scaling Group (EC2 instances running Docker)
- ECR repository for container images
- Optional RDS PostgreSQL (when enabled in tfvars)
- Terraform state backend:
  - S3 bucket for state
  - DynamoDB table for state lock

---

## Variables
Infrastructure configuration is driven by `env/dev.tfvars`.

Key values include (examples):
- `project_name`
- `enable_rds`
- `image_tag` (passed from CD pipeline; typically commit SHA)

---

## Remote state backend
Terraform uses an S3 bucket backend with DynamoDB locking. The CI/CD pipeline ensures the backend exists before running Terraform, then writes `infra/backend.hcl` (not committed) and runs:

- `terraform init -backend-config=backend.hcl`

---

## Deployment model
- The pipeline builds a Docker image and pushes it to ECR.
- Terraform config references `image_tag` to define which image EC2 instances should run.
- After deployment, the pipeline starts an **ASG Instance Refresh** to roll out the new image version.

---

## How to run locally (optional)
From `infra/`:

```bash
terraform init -backend-config=backend.hcl
terraform plan -var-file=env/dev.tfvars -var="image_tag=local"
terraform apply -var-file=env/dev.tfvars -var="image_tag=local"

