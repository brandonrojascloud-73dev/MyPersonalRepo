# GitHub Actions Self-Hosted Runners: Ephemeral Architecture at Scale

## Context

Our engineering organization needed CI/CD infrastructure that could:
- Scale from 10 to 500+ concurrent jobs
- Run in isolated, clean environments every time
- Support multiple AWS accounts and regions
- Reduce runner costs by 60%+ vs. GitHub-hosted

## The Architecture

### Decision: Ephemeral Runners with Packer + Auto Scaling

```
GitHub Webhook
    │
    ▼
Lambda (runner registration)
    │
    ├── Creates JIT token
    │
    ▼
Auto Scaling Group
    │
    ├── Launch Template (Packer AMI)
    │   ├── Ubuntu 22.04 base
    │   ├── Pre-installed tools (aws-cli, docker, kubectl, terraform)
    │   ├── CloudWatch agent
    │   └── SSM agent
    │
    ├── Min: 0, Max: 500
    ├── Scale policy: queue depth based
    └── Instance lifecycle: run → deregister → terminate
```

### Why Ephemeral?

| Approach | Pros | Cons |
|----------|------|------|
| Long-lived EC2 | Simple, fast startup | State drift, security risk, cost |
| GitHub-hosted | Zero ops | Expensive, limited customization |
| **Ephemeral** | **Clean every run, auto-scale, 60% cheaper** | **Complex setup, cold start** |

We chose ephemeral because security isolation and cost justified the complexity.

## Key Implementation Details

### 1. Packer Image Pipeline

```hcl
# packer/runner.pkr.hcl
source "amazon-ebs" "runner" {
  ami_name      = "gha-runner-{{timestamp}}"
  instance_type = "c5.xlarge"
  source_ami_filter {
    filters = {
      name = "ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"
    }
    owners      = ["099720109477"]  # Canonical
    most_recent = true
  }
}

build {
  sources = ["source.amazon-ebs.runner"]
  
  provisioner "shell" {
    inline = [
      "sudo apt-get update && sudo apt-get install -y ...",
      "curl -fsSL https://get.docker.com | sh",
      "aws s3 cp s3://runner-config/tools.tar.gz /opt/ && tar xzf /opt/tools.tar.gz",
      "sudo systemctl enable cloudwatch-agent"
    ]
  }
}
```

Image rebuilt weekly via scheduled pipeline. Old AMIs deregistered after 14 days.

### 2. Runner Registration (Lambda)

```python
# Simplified registration flow
def handle_webhook(event):
    # 1. Validate webhook signature
    validate_signature(event)
    
    # 2. Create JIT registration token
    token = github_api.create_registration_token(
        org="our-org",
        labels=["self-hosted", "linux", "x64"]
    )
    
    # 3. Pass token via SSM Parameter (encrypted)
    ssm.put_parameter(
        Name=f"/gha-runner/{instance_id}/token",
        Value=token,
        Type="SecureString",
        Tier="Standard"
    )
    
    # 4. Trigger ASG scale-up
    asg.set_desired_capacity(current + 1)
```

### 3. Runner Lifecycle

```
Instance boots
    │
    ├── SSM agent fetches registration token
    ├── Runner starts: ./config.sh --url ... --token $TOKEN --ephemeral
    ├── Executes job
    ├── Runner auto-deregisters (--ephemeral flag)
    ├── CloudWatch reports "idle"
    └── ASG terminates instance (scale-in)
```

The `--ephemeral` flag is critical: the runner accepts exactly one job, then exits. No state persists between runs.

### 4. Multi-Account Architecture

```
Central Hub Account
    │
    ├── GitHub webhook receiver (Lambda)
    ├── Packer image pipeline
    ├── Shared AMI via RAM
    │
    ├── Target Account A
    │   └── ASG + Launch Template (shared AMI)
    │
    ├── Target Account B
    │   └── ASG + Launch Template (shared AMI)
    │
    └── Target Account C
        └── ASG + Launch Template (shared AMI)
```

AMIs shared via AWS Resource Access Manager (RAM). Each account runs its own ASG but uses the same Packer-built image.

## Scaling Behavior

Real-world scaling during peak hours:

```
08:00  - 12 runners (baseline)
09:30  - 45 runners (morning PR wave)
11:00  - 120 runners (merge train)
14:00  - 85 runners (afternoon)
17:00  - 200 runners (release pipeline)
18:30  - 35 runners (winding down)
```

Scale-up latency: ~90 seconds (instance boot + runner registration). Mitigated by maintaining a small warm pool.

## Cost Impact

| Metric | GitHub-Hosted | Self-Hosted Ephemeral |
|--------|--------------|----------------------|
| Monthly cost (500 jobs/day) | ~$15,000 | ~$5,800 |
| Cost per job | $1.00 | $0.39 |
| Cold start time | 0s | ~90s |
| Customization | Limited | Full |

**61% cost reduction** while improving security (ephemeral isolation) and flexibility (custom tooling).

## Challenges Solved

1. **Cold start**: 90-second boot time unacceptable for small jobs. Solution: maintain 5-instance warm pool that handles immediate jobs while ASG scales.

2. **Token security**: Registration tokens in user data are visible. Solution: SSM Parameter Store with instance role access only.

3. **Orphaned runners**: Instances that crash without deregistering. Solution: Lambda cleanup job runs hourly, removes stale runners via GitHub API.

4. **AMI drift**: Images becoming outdated. Solution: weekly Packer rebuild, automated AMI rotation, ASG launch template updated via CI.

## Why This Matters

This architecture demonstrates:
- **Infrastructure as code**: Packer, Terraform, Lambda — all version-controlled
- **Security-first design**: Ephemeral isolation, encrypted tokens, no persistent state
- **Cost engineering**: 61% savings through right-sizing and auto-scaling
- **Multi-account patterns**: Central hub with shared AMIs via RAM
- **Operational maturity**: Monitoring, alerting, automated cleanup

It's the kind of platform engineering work that enables hundreds of developers to ship code reliably every day.