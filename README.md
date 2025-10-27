# DevOps Skills

Community repository of DevOps-focused skills for [Claude Code](https://claude.com/claude-code).

## Available Skills

### iac-terraform
**Infrastructure as Code with Terraform and Terragrunt**

Use for creating, validating, troubleshooting, and managing Terraform configurations, modules, and state. Includes state inspection tools, module validators, and comprehensive troubleshooting guides.

### k8s-troubleshooter
**Systematic Kubernetes troubleshooting and incident response**

Diagnose pod failures, cluster issues, performance problems, and production incidents. Features cluster health checks, pod diagnostics, and structured incident response playbooks.

### aws-cost-finops
**AWS cost optimization and FinOps workflows**

Find unused resources, analyze Reserved Instance opportunities, detect cost anomalies, rightsize instances, evaluate Spot instances, and implement FinOps best practices.

**Features:**
- 🔍 6 automated analysis scripts (find waste, analyze RIs, detect old generations, evaluate Spot, rightsize resources, detect anomalies)
- 📊 Comprehensive reference guides (best practices, service alternatives, FinOps governance)
- 📝 Monthly cost report template
- 💰 Proven to find real cost savings on first run
- ⚡ Full integration with AWS APIs (EC2, RDS, EBS, S3, CloudWatch, Cost Explorer)

### ci-cd
CI/CD pipeline design, optimization, security, and troubleshooting. Create workflows, optimize build performance, implement caching, secure pipelines, and debug issues across GitHub Actions, GitLab CI, and other platforms.

## Installation

Add the marketplace:
```bash
/plugin marketplace add https://github.com/ahmedasmar/devops-claude-skills
```

Install skills:
```bash
/plugin install iac-terraform@devops-skills
/plugin install k8s-troubleshooter@devops-skills
/plugin install aws-cost-finops@devops-skills
/plugin install ci-cd@devops-skills
```

## Usage

Once installed, use these skills through Claude Code by describing what you need:

**AWS Cost Optimization:**
- "Find unused AWS resources that are costing me money"
- "Analyze my EC2 instances for Reserved Instance opportunities"
- "What's the cheapest way to store infrequently accessed data in S3?"
- "Help me set up a monthly AWS cost review process"
- "Detect cost anomalies in my AWS spending"

**Terraform:**
- "Help me create a reusable Terraform module for VPC"
- "Review my Terraform state for drift"
- "Troubleshoot this Terraform error"

**Kubernetes:**
- "This pod is in CrashLoopBackOff, help me diagnose it"
- "Check the health of my Kubernetes cluster"
- "Help me troubleshoot this deployment"

## Contributing

To contribute a new DevOps skill:

1. Fork this repository
2. Create a new directory with your skill name (lowercase, hyphenated)
3. Add `.claude-plugin/plugin.json` manifest
4. Add `skills/SKILL.md` with proper frontmatter
5. Update `.claude-plugin/marketplace.json` to include your skill
6. Submit a pull request

## License

MIT
