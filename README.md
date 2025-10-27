# DevOps Skills

Community repository of DevOps-focused skills for [Claude Code](https://claude.com/claude-code).

## Available Skills

### iac-terraform
Infrastructure as Code with Terraform and Terragrunt. Use for creating, validating, troubleshooting, and managing Terraform configurations, modules, and state.

### k8s-troubleshooter
Systematic Kubernetes troubleshooting and incident response. Diagnose pod failures, cluster issues, performance problems, and production incidents.

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
/plugin install ci-cd@devops-skills
```

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
