
# Architecture Overview (ASG + ALB + RDS)

## Summary
This project deploys a containerised web application to AWS using a CI/CD pipeline (GitHub Actions) and Infrastructure as Code (Terraform). The runtime architecture uses **EC2 Auto Scaling Group (ASG)** behind an **Application Load Balancer (ALB)**. The application container image is stored in **Amazon ECR**. Data persistence is provided by **Amazon RDS (PostgreSQL)** when enabled.

Region: **eu-north-1**

---

## High-level architecture

**Developer workflow**
1. Developer pushes code to GitHub (main branch)
2. CI runs automated tests + security scanning
3. CD builds a Docker image, pushes it to ECR, applies Terraform, and triggers an ASG Instance Refresh to roll out the new version

**Runtime components**
- **ALB**: public entrypoint; routes HTTP traffic to EC2 instances in the ASG
- **ASG / Launch Template**: EC2 instances run Docker and pull the application image from ECR
- **ECR**: stores the built container images (tagged by commit SHA and `latest`)
- **RDS (PostgreSQL)**: optional database used by the application
- **SSM Parameter Store**: stores application configuration such as `DATABASE_URL` (where implemented)
- **Terraform remote state backend**:
  - **S3 bucket** stores terraform state
  - **DynamoDB table** stores state lock to prevent concurrent applies

---

## Data flow
1. User sends HTTP request to ALB DNS name.
2. ALB forwards traffic to a healthy EC2 instance in the ASG.
3. EC2 instance runs the Docker container.
4. Application reads config (e.g., `DATABASE_URL`) and connects to RDS if enabled.
5. Response is returned via ALB to the user.

---

## Deployment flow (CI/CD)

### CI (Continuous Integration)
CI validates changes before deployment:
- Unit tests using `pytest`
- Security scanning using `bandit`

If CI fails, CD does not proceed.

### CD (Continuous Deployment)
CD automates:
- Ensuring Terraform backend exists (S3 + DynamoDB)
- Terraform `init` and `apply` to create/update infrastructure
- Docker build + push to ECR
- ASG Instance Refresh to roll out the new image across instances

---

## Why this architecture
- **ASG + ALB** is a simple, production-relevant pattern:
  - ALB provides a stable entrypoint and health checks
  - ASG provides scaling and self-healing (replace unhealthy instances)
- **Docker** ensures consistent runtime across environments
- **Terraform** enables repeatable environment creation and versioned infrastructure changes
- **Remote Terraform state** supports safer automation and prevents state corruption via locking

---

## Notes / limitations
- In a larger production system, this design could be extended with:
  - multiple environments (dev/staging/prod)
  - HTTPS via ACM + ALB listener rules
  - blue/green deployments or canary releases
  - monitoring/alerting (CloudWatch metrics/alarms, logs, dashboards)
