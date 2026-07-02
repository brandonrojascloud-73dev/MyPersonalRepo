# Brandon Fernandez Rojas — Senior DevOps, Platform & Site Reliability Engineer

Senior DevOps Engineer with 9+ years designing, automating, and operating enterprise cloud and hybrid infrastructure. Deep expertise across AWS, Linux, Kubernetes, Terraform, GitLab CI/CD, GitHub Actions, and GitOps, with a proven track record of improving production reliability, tightening security posture, and eliminating toil through automation.

Live portfolio: [brandonrojascloud-73dev.github.io/MyPersonalRepo](https://brandonrojascloud-73dev.github.io/MyPersonalRepo/)

## Contact

- Location: Zapopan, Jalisco, Mexico
- Phone: [+52 332 822 1329](tel:+523328221329)
- Email: [brandon.rojas.cloud@gmail.com](mailto:brandon.rojas.cloud@gmail.com)
- LinkedIn: [linkedin.com/in/brandon-fernandez-rojas](https://www.linkedin.com/in/brandon-fernandez-rojas)
- GitHub: [github.com/brandonrojascloud-73dev](https://github.com/brandonrojascloud-73dev)

## Core Technical Skills

### Cloud & Platforms
AWS (EC2, VPC, IAM, S3, RDS, ELB, EBS, CloudTrail, Systems Manager, Secrets Manager, KMS), Azure, Cloudflare

### IaC & Automation
Terraform, CloudFormation, Ansible, Packer, AWS Systems Manager, Python, Bash, PowerShell

### Containers & Orchestration
Kubernetes, Amazon EKS, Amazon ECS, Docker, FluxCD, GitOps, Helm-style workflows

### CI/CD & SCM
GitLab CI/CD, GitHub Actions, Jenkins, Git workflows, pipeline design and troubleshooting

### Observability
Datadog, Grafana, CloudWatch, Elasticsearch, PagerDuty, Icinga

### Security & Compliance
IAM, RBAC, KMS, Secrets Manager, SOPS, SELinux, CrowdStrike Falcon, VPN, Security Groups

### Operating Systems
RHEL, Ubuntu, SUSE, Fedora, Windows Server, LDAP integration

### Methodologies
SRE practices, incident response, RCA, GitOps, Agile, ITIL, DevSecOps

## Professional Experience

### Senior DevOps Engineer — Teradata
May 2023 – Present

- Own end-to-end observability across Datadog, Elasticsearch, and Grafana, designing dashboards, log pipelines, and metric-based alerts that improve mean-time-to-detect on production services.
- Operate and evolve CI/CD and GitOps delivery using GitLab CI, GitHub Actions, Terraform, and FluxCD to standardize infrastructure provisioning and application rollouts.
- Maintain reliability of EKS and ECS workloads, troubleshooting Docker and Kubernetes issues such as probes, rollouts, and resource pressure to sustain high availability.
- Harden secrets management using SOPS with AWS KMS and manage backup and recovery workflows across AWS and Azure to meet security and compliance requirements.
- Partner with global engineering teams and customers on PoCs, KTs, and implementation sessions, delivering resilient DevOps solutions and clear operational runbooks.

### Senior Infrastructure Cloud Engineer — IBM
Jun 2021 – May 2023

- Provisioned and operated AWS infrastructure across EC2, S3, VPN tunnels, ELBs, and ECS clusters supporting enterprise workloads.
- Codified infrastructure with Terraform and automated patching and configuration through AWS Systems Manager and Ansible for consistent, version-controlled builds.
- Enforced cloud security posture via IAM policy design, CloudTrail auditing, and SOPS/KMS encryption, aligning environments with the AWS Well-Architected Framework.
- Integrated LDAP-based centralized identity across Linux workloads and administered access controls for hybrid teams.
- Monitored and resolved infrastructure incidents using CloudWatch, Grafana, and Icinga; deployed and troubleshot CrowdStrike Falcon and SaltStack agents.
- Developed automation in Python, PowerShell, and SSM Run Command; managed GitHub Actions and GitLab CI/CD pipelines and supported RDS/Aurora operations.

### Senior Cloud Engineer — HCL
Jan 2020 – Jun 2021

- Built and maintained multi-cloud infrastructure across AWS, Azure, and Cloudflare for high-availability workloads.
- Automated provisioning and configuration with Terraform, Ansible, and Packer; administered RHEL, SLES, Fedora, and Ubuntu fleets with LDAP integration.
- Operated networking and security including VPC, firewalls, security groups, and NACLs, and implemented GitOps workflows with FluxCD for Kubernetes application delivery.
- Tuned platform performance and reliability using Datadog, Grafana, Elasticsearch, and PagerDuty; optimized workloads across EKS and ECS.

### Linux Systems Engineer — TATA Consultancy Services
Jan 2019 – Jan 2020

- Supported Linux and Windows server fleets, applying cloud security best practices, IAM, and encryption standards.
- Provisioned VPCs, storage, and compute using IaC and developed GitLab CI/CD pipelines for cloud-native deployment workflows.
- Monitored AWS infrastructure including IAM, EC2, CloudWatch, and CloudTrail; triaged vulnerabilities and managed SELinux security policies.
- Reviewed Python code and authored Bash automation to reduce repetitive operational work.

### System Administrator — ITSystem
Jan 2018 – Feb 2019

- Administered Linux and Windows servers with hands-on troubleshooting; managed AWS and VMware infrastructure monitoring via Datadog, Grafana, and CloudWatch.
- Remediated vulnerabilities using Ansible; upgraded and decommissioned servers via AWS SSM and Ansible; scripted operations in Bash and Python.
- Owned RHEL permissions administration, WebSphere support, and disaster recovery, backup, and restore procedures.

### Senior Operations Analyst — Cognizant
Sep 2016 – Jan 2018

- Delivered remote technical support for VPN connectivity, Windows systems, and ERP-related incidents; handled service requests through ServiceNow, including Active Directory user administration.
- Documented resolutions, escalated critical issues to internal IT teams, and contributed to knowledge-transfer materials.

## Certifications & Education

### Certifications
- AWS Certified DevOps Engineer – Professional
- AWS Certified SysOps Administrator – Associate
- AWS Certified Cloud Practitioner
- Datadog SRE 101 Fundamentals
- Agile Certified

### Education
Computer Systems Technologist — Centro de Enseñanza Técnica Industrial (CETI)

### Languages
- Spanish: Native
- English: Advanced

## Repository Purpose

This repository hosts a professional interactive resume and portfolio using a static HTML page deployed through GitHub Pages. The deployment pipeline is intentionally simple and production-oriented:

- Static HTML/CSS with no runtime framework dependency
- GitHub Actions workflow for automated deployment on every push to `main`
- GitHub Pages hosting
- README aligned with the published portfolio content

## Deployment

The site is deployed automatically with GitHub Actions using `.github/workflows/deploy.yml`.

```mermaid
flowchart LR
  A[Push to main] --> B[GitHub Actions]
  B --> C[Upload Pages artifact]
  C --> D[Deploy to GitHub Pages]
  D --> E[Live portfolio]
```

## License

MIT © Brandon Fernandez Rojas
