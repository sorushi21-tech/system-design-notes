# Infrastructure as Code and DevOps

Infrastructure as Code and DevOps practices make infrastructure and deployments repeatable, reviewable, and safer. They turn manual operations into version-controlled workflows.

## 1. Why Infrastructure Automation Matters

Manual infrastructure changes cause problems:

- Different environments drift apart.
- Nobody remembers exact setup steps.
- Production changes are hard to review.
- Rollback is unclear.
- Security rules become inconsistent.
- Disaster recovery takes longer.

Infrastructure as Code solves this by storing infrastructure definitions in code.

Benefits:

- Reproducible environments
- Pull request review
- Audit history
- Easier rollback
- Consistent dev/staging/prod setup
- Less manual error

---

## 2. Declarative vs Imperative

### Declarative

You describe the desired final state.

Examples:

- Terraform
- CloudFormation
- Kubernetes YAML

Example idea:

```text
There should be three app instances behind a load balancer.
```

The tool figures out how to reach that state.

### Imperative

You describe exact steps.

Examples:

- Shell scripts
- CLI scripts
- Some Ansible workflows

Example idea:

```text
Create VM, install package, copy config, restart service.
```

Imperative automation is flexible, but long scripts can become hard to reason about.

---

## 3. Terraform

Terraform is a popular Infrastructure as Code tool for managing AWS infrastructure.

Core concepts:

| Concept | Meaning |
| --- | --- |
| Provider | Plugin for AWS services and related platforms such as Kubernetes |
| Resource | Infrastructure object to create/manage |
| Data source | Read existing infrastructure |
| Variable | Input value |
| Output | Exported value |
| Module | Reusable infrastructure package |
| State | Mapping between code and real resources |

### Terraform Workflow

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
```

Meaning:

- `init`: download providers and initialize backend
- `fmt`: format files
- `validate`: check syntax and basic validity
- `plan`: preview changes
- `apply`: make changes

### State

Terraform state is critical. It tells Terraform which real resources belong to which code blocks.

State contains sensitive information sometimes, so protect it.

Best practices:

- Use remote state.
- Enable state locking.
- Encrypt state.
- Restrict access.
- Back it up.
- Do not commit local state files.

Remote state examples:

- AWS S3 with DynamoDB locking
- Terraform Cloud

For AWS teams, a common backend is:

```hcl
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "prod/app/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

The S3 bucket stores state. DynamoDB provides locking so two engineers or pipelines do not apply changes at the same time.

### Modules

Modules package reusable infrastructure.

Good module examples:

- VPC module
- Kubernetes cluster module
- Database module
- Service deployment module
- ECS service module
- RDS/Aurora module
- SQS/SNS/EventBridge module
- IAM role module

Good module design:

- Clear inputs
- Clear outputs
- Few unnecessary options
- Good defaults
- Versioned releases
- Documentation

### Drift

Drift means real infrastructure differs from code.

Causes:

- Manual console changes
- Emergency fixes
- Failed applies
- External automation

Control drift by:

- Avoiding manual changes
- Running scheduled plans
- Reconciling emergency changes back into code
- Alerting on unexpected changes

---

## 4. CloudFormation

CloudFormation is AWS-native Infrastructure as Code.

Features:

- YAML or JSON templates
- AWS-managed stack state
- Change sets
- Nested stacks
- StackSets

Strengths:

- Deep AWS integration
- No separate state backend
- Works well with AWS governance

Trade-offs:

- AWS-only
- Verbose templates
- Some workflows feel slower than Terraform

Use CloudFormation when the organization is strongly AWS-native and wants provider-managed stack state.

---

## 5. Ansible

Ansible is used for automation and configuration management.

It is agentless and usually connects over SSH.

Core concepts:

| Concept | Meaning |
| --- | --- |
| Inventory | Target machines |
| Playbook | Automation definition |
| Task | One action |
| Role | Reusable structure |
| Handler | Action triggered by change |
| Template | Generated config file |

Good for:

- OS configuration
- Package installation
- Application deployment on VMs
- Config file management
- Operational tasks

Less ideal for:

- Complex cloud resource lifecycle
- Replacing Terraform for long-lived infrastructure

Common pairing:

```text
Terraform creates infrastructure.
Ansible configures servers.
```

---

## 6. GitOps

GitOps uses Git as the source of truth for desired runtime state.

Typical flow:

```text
Developer opens pull request
  -> CI validates manifests
  -> Pull request is merged
  -> GitOps controller sees change
  -> Cluster is reconciled to match Git
```

Common tools:

- Argo CD
- Flux

On AWS, GitOps is most common with EKS. For ECS, teams more commonly use CI/CD pipelines that update ECS task definitions and services.

Benefits:

- Pull request review for deployments
- Clear audit history
- Easy rollback using Git revert
- Drift detection
- CI does not need broad cluster credentials

Works best with:

- Declarative manifests
- Immutable images
- Environment-specific overlays
- Safe secret management
- Tested rollback process

---

## 7. CI/CD

### Continuous Integration

CI validates every change.

Typical steps:

1. Checkout code
2. Install dependencies
3. Build application
4. Run tests
5. Run static analysis
6. Run security scan
7. Build artifact
8. Publish artifact

The artifact should be immutable.

Examples:

- Container image
- JAR file
- Binary
- NPM package

### Continuous Delivery

The artifact is always ready to deploy, but production release may require approval.

### Continuous Deployment

Every passing change can automatically go to production.

This requires:

- Strong automated tests
- Good observability
- Fast rollback
- Safe deployment strategy
- Small changes

### Build Once, Promote

Bad pattern:

```text
Build app for dev
Build app again for staging
Build app again for prod
```

Good pattern:

```text
Build image my-service:1.4.2 once
Deploy same image to dev, staging, and prod
Change only environment configuration
```

AWS-oriented artifact flow:

```text
Build Spring Boot app
  -> build Docker image
  -> push image to ECR
  -> deploy same image tag to dev
  -> promote same image tag to staging
  -> promote same image tag to prod
```

Common AWS CI/CD services:

- CodeBuild for builds
- CodePipeline for orchestration
- CodeDeploy for deployment strategies
- ECR for container images
- GitHub Actions or Jenkins if the team already standardizes there

---

## 8. Deployment Strategies

### Rolling Deployment

Gradually replaces old instances with new ones.

Pros:

- Simple
- Resource efficient
- Common default

Cons:

- Old and new versions run together
- Rollback can take time
- Requires backward compatibility

### Blue-Green Deployment

Two environments exist:

```text
Blue = current production
Green = new version
```

Traffic switches from Blue to Green after validation.

Pros:

- Fast rollback
- Clean traffic switch
- Production-like validation before release

Cons:

- Higher temporary cost
- Database changes must be backward compatible
- Stateful systems are harder

### Canary Release

A small percentage of traffic receives the new version first.

Example:

```text
1% -> 5% -> 10% -> 25% -> 50% -> 100%
```

Monitor at each step:

- Error rate
- Latency
- Saturation
- Logs
- Business metrics

Canary releases work best with automated rollback.

### Feature Flags

Feature flags separate deployment from release.

Use cases:

- Dark launch
- Kill switch
- A/B test
- Gradual rollout
- Tenant-specific enablement

Flag management rules:

- Every flag should have an owner.
- Every flag should have a purpose.
- Every temporary flag should have an expiry date.
- Remove old flags after rollout.

---

## 9. Autoscaling

Autoscaling changes capacity based on demand.

### Reactive Scaling

Scales after metrics cross a threshold.

Examples:

- CPU > 70%
- Memory > 80%
- Queue depth > 1000
- Requests per pod > 500

### Predictive Scaling

Scales before expected traffic arrives.

Good for:

- Daily traffic cycles
- Weekly business patterns
- Known launch events

### Scale-to-Zero

Serverless platforms can scale to zero when idle.

Pros:

- Saves cost
- Good for low-traffic workloads

Cons:

- Cold starts
- Not ideal for strict low-latency services

### Important Scaling Rule

Scaling the application does not fix every bottleneck.

Check dependencies:

- Database connections
- Cache capacity
- Queue throughput
- External API limits
- Network bandwidth
- Lock contention

---

## 10. Load Balancing

Load balancers distribute traffic across healthy targets.

### Layer 4 Load Balancer

Works at TCP/UDP level.

Good for:

- High throughput
- Low latency
- Non-HTTP traffic

### Layer 7 Load Balancer

Works at HTTP/HTTPS level.

Can route by:

- Host
- Path
- Header
- Cookie

Can also provide:

- TLS termination
- Redirects
- WAF integration
- Access logs
- Sticky sessions

### Health Checks

A load balancer health check decides whether a target receives traffic.

Good health checks:

- Are fast
- Reflect readiness
- Avoid expensive dependency checks
- Fail when the target cannot serve real traffic

### Connection Draining

Connection draining lets in-flight requests finish before a target is removed.

Useful during:

- Deployments
- Scale-in
- Maintenance
- Instance replacement

---

## 11. Secrets and Configuration

Do not commit secrets to Git.

Use secret managers:

- AWS Secrets Manager
- AWS SSM Parameter Store
- HashiCorp Vault
- Kubernetes External Secrets

Best practices:

- Encrypt secrets at rest.
- Restrict access by role.
- Rotate secrets.
- Audit secret access.
- Separate secrets by environment.
- Do not print secrets in logs.

---

## 12. Observability in DevOps

Deployment automation is incomplete without observability.

You need:

- Metrics
- Logs
- Traces
- Alerts
- Dashboards
- Runbooks

Golden signals:

- Latency
- Traffic
- Errors
- Saturation

Deployment should answer:

- Did error rate increase?
- Did latency increase?
- Did CPU/memory saturation increase?
- Did business metrics change?
- Should we continue rollout or rollback?

---

## 13. Production Checklist

- Infrastructure is defined in code.
- Infrastructure changes go through pull requests.
- Remote state is locked and encrypted.
- CI produces immutable artifacts.
- Same artifact is promoted across environments.
- Secrets are outside Git.
- Deployment strategy is chosen per service risk.
- Rollback process is tested.
- Health checks are meaningful.
- Autoscaling is tested under load.
- Load balancer draining is configured.
- Manual console changes are reconciled back to code.
- Cost tags or labels are applied.
- Alerts and dashboards exist before production launch.
