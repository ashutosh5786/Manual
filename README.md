# Infrastructure Architecture Overview

![Architectual Diagram](./architecture%20.png)

## High-Level Architecture Explanation

- **Frontend**: ReactJS app hosted on Vercel.
- **Backend**: AWS API Gateway (HTTP API) exposed publicly, connected to VPC Link.
- **VPC Link**: Connects API Gateway to an internal ALB.
- **Internal ALB**: Routes traffic to ECS services deployed in private subnets.
- **ECS Services**: Backend microservices (eShop, Pharmacy & Doctor) deployed inside ECS, running on Fargate/EC2.
- **Communication**:
  - Frontend to Backend: Via API Gateway.
  - Backend to Backend: Via internal ALB using path-based routing.

## Decisions and Tradeoffs

| Decision               | Tradeoff                           | Rationale                                                          |
| :--------------------- | :--------------------------------- | :----------------------------------------------------------------- |
| API Gateway + VPC Link | Slight cost and config overhead    | Strong security and scalability                                    |
| ECS for Backend        | Slightly higher cost (for Fargate) | No server management, easy scaling, can be switch to EC2 if needed |
| Internal ALB           | -                                  | Secure private routing and health checks                           |
| Private Subnets        | Use of NAT Gateway increases bills | Maximum backend security                                           |

---

## Database

- **Database**: AWS RDS (MySQL/Postgres) deployed in private subnets.
- **Connectivity**:
  - ECS Tasks (running in private subnets) securely connect to RDS via private VPC networking.
  - No public access to RDS.
  - Security Groups:
    - ECS SG → RDS SG (port 3306/5432 open)
    - RDS SG does not allow traffic from the internet.
- **Credentials Management**:
  - RDS credentials stored in AWS Secrets Manager (or SSM Parameter Store).
  - Applications fetch secrets dynamically without hardcoding DB passwords.

---

## Monitoring, Alerting & SLOs

- **Tools**: Primary monitoring via CloudWatch.
- **Key Metrics**:
  - API Gateway latency (p95, p99)
  - API Gateway 4xx/5xx error rates
  - ALB Target Group health
  - ECS Task CPU and memory utilization
- **Alerts**:
  - High 5xx errors
  - ECS CPU > 70% or Memory > 80%
  - Target Group unhealthy hosts > threshold
- **SLOs**:
  - 99.9% API availability
  - 95% p95 latency < 500ms
  - ECS Task failure rate < 0.1%

---

## Infrastructure as Code (IaC)

- **Tool**: Terraform (preferred)
- **Structure**:

  ```bash
  /terraform
    /modules
      /ecs-service
      /alb
      /vpc-link
      /api-gateway
    /envs
      /prod
      /staging
      /dev
  ```

  ***

## Security and Compliance (Healthcare Context)

- Encrypt data at rest (EBS, RDS) and in transit (TLS).

- Use Secrets Manager or SSM Parameter Store with KMS.

- Least privilege IAM policies.

- Full audit trails (CloudTrail enabled across all regions).

- WAF protection at API Gateway.

---

## FinOps and Cost Optimization

- **Short-term:**

  - Right-size ECS task CPU/memory.

  - Enable API Gateway caching.

  - ECS Auto Scaling based on request/CPU metrics.

  - Budget alerts for early detection.

- **Mid-term:**

  - Purchase Savings Plans for ECS/Fargate.

  - Spot Instances for non-production environments.

- **Long-term:**

  - Monthly cloud spend analysis.

---

## CI/CD Pipeline Overview

### Project Structure

Only deploy.yml is present in this repository as cicd.yml

```bash
  / (root)
  ├── index.php             # Main PHP backend file
  ├── Dockerfile            # Docker image build instructions
  ├── composer.json         # PHP dependencies (PHPUnit)
  ├── phpunit.xml           # PHPUnit configuration
  ├── /tests                # Basic test case
  │    └── HealthCheckTest.php
  ├── /deploy               # Deployment files
  │    └── ecs-task-def.json
  └── /.github
      └── /workflows
          └── deploy.yml   # GitHub Actions CI/CD workflow
```

### How the CI/CD Pipeline Works

**1. Trigger:** On push to the main branch.

**2. Lint & Test:**

- Install PHP dependencies.

- Run PHPUnit unit tests.

**3. Build & Push Docker Image:**

- Build the PHP backend image using Docker.

- Push the image to Amazon ECR.

**4. Update Task Definition:**

- Inject the newly built image into ECS task definition.

**5. Deploy to ECS:**

- Update the ECS service to use the latest task definition.

- Force new deployment and wait for service stability.

## GitHub Actions Workflow

- Configure AWS Credentials using GitHub OIDC and IAM Role.

- CloudWatch Logs are configured using awslogs driver.

- Service Update automatically triggers a new ECS deployment.

---
