# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a **Claude Code Skills Marketplace** repository containing DevOps-focused skills. It follows the Claude Code plugin/skill architecture where each skill is a self-contained directory with reference materials, scripts, and structured guidance.

## Repository Structure

```
devops-claude-skills/
├── .claude-plugin/
│   └── marketplace.json          # Marketplace registry (lists all available skills)
├── iac-terraform/                # Skill: Terraform & Terragrunt
│   ├── .claude-plugin/
│   │   └── plugin.json          # Skill metadata
│   └── skills/
│       ├── SKILL.md             # Main skill content (loaded when skill invoked)
│       ├── references/          # Supporting documentation
│       ├── scripts/             # Diagnostic/utility scripts
│       └── assets/              # Templates and resources
└── k8s-troubleshooter/          # Skill: Kubernetes troubleshooting
    ├── .claude-plugin/
    │   └── plugin.json
    └── skills/
        ├── SKILL.md
        ├── references/
        └── scripts/
```

## Architecture Principles

### 1. Skills as Self-Contained Modules
Each skill directory is completely independent and contains:
- **SKILL.md**: Main instructional content (the "brain" of the skill)
- **references/**: Detailed supporting documentation (troubleshooting guides, best practices)
- **scripts/**: Python automation scripts for diagnostics and validation
- **assets/**: Templates, examples, and reusable resources
- **.claude-plugin/plugin.json**: Metadata (name, description, version, author)

### 2. Two-Layer Documentation Pattern
Skills use a two-layer documentation approach:
- **SKILL.md**: Workflow-oriented guidance with decision trees, quick references, and "when to use" sections
- **references/*.md**: Deep-dive documentation for specific topics (e.g., `troubleshooting.md`, `best_practices.md`)

This pattern allows SKILL.md to stay focused on workflows while references provide comprehensive detail when needed.

### 3. Diagnostic Scripts Pattern
Each skill includes Python scripts that follow a common pattern:
- Accept relevant parameters (path, namespace/pod, etc.)
- Perform automated checks and analysis
- Provide structured output with actionable recommendations
- Can be run standalone or integrated into workflows

## Available Skills

### iac-terraform
**Purpose**: Terraform and Terragrunt infrastructure as code workflows

**Key Components**:
- `skills/SKILL.md`: Comprehensive Terraform/Terragrunt workflows including module development, state management, and troubleshooting
- `skills/references/best_practices.md`: Project structure, state management, module design, security practices, CI/CD patterns
- `skills/references/troubleshooting.md`: Common issues (state locks, drift, provider errors, resource issues, performance)
- `skills/scripts/inspect_state.py`: Terraform state inspection and drift detection
- `skills/scripts/validate_module.py`: Module validation against best practices
- `skills/assets/templates/MODULE_TEMPLATE.md`: Complete module structure template

**Workflow Pattern**: Decision-tree based (Is this reusable? → Module vs Environment Config)

### k8s-troubleshooter
**Purpose**: Systematic Kubernetes troubleshooting and incident response

**Key Components**:
- `skills/SKILL.md`: Systematic troubleshooting workflow from context gathering to post-incident
- `skills/references/common_issues.md`: Detailed issue library (ImagePullBackOff, CrashLoopBackOff, OOMKilled, node issues, networking, storage, RBAC)
- `skills/references/incident_response.md`: Structured incident response framework with severity levels and playbooks
- `skills/scripts/cluster_health.py`: Cluster-wide health check (nodes, system pods, failures)
- `skills/scripts/diagnose_pod.py`: Pod-level diagnostics with actionable recommendations

**Workflow Pattern**: Systematic triage (Gather Context → Initial Triage → Deep Dive → Root Cause → Remediation → Verify)

## Contributing New Skills

When adding a new DevOps skill to this marketplace:

1. **Create skill directory structure**:
   ```
   new-skill-name/
   ├── .claude-plugin/
   │   └── plugin.json
   └── skills/
       ├── SKILL.md
       ├── references/
       ├── scripts/
       └── assets/
   ```

2. **Write plugin.json** with:
   ```json
   {
     "name": "skill-name",
     "description": "One-line description",
     "version": "1.0.0",
     "author": {
       "name": "Your Name"
     }
   }
   ```

3. **Structure SKILL.md** with:
   - YAML frontmatter (name, description)
   - "When to Use This Skill" section
   - Core workflows with decision trees or systematic processes
   - Quick reference commands
   - Links to reference documentation
   - Best practices summary

4. **Update .claude-plugin/marketplace.json** to register the new skill:
   ```json
   {
     "name": "skill-name",
     "source": "./skill-name",
     "description": "Brief description"
   }
   ```

5. **Update root README.md** to list the new skill under "Available Skills"

## Testing Skills

Skills can be tested by:

1. **Installing the marketplace locally**:
   ```bash
   /plugin marketplace add /path/to/devops-claude-skills
   ```

2. **Installing individual skills**:
   ```bash
   /plugin install iac-terraform@devops-skills
   /plugin install k8s-troubleshooter@devops-skills
   ```

3. **Testing script functionality**:
   ```bash
   # Terraform scripts
   python3 iac-terraform/skills/scripts/inspect_state.py /path/to/terraform
   python3 iac-terraform/skills/scripts/validate_module.py /path/to/module

   # Kubernetes scripts
   python3 k8s-troubleshooter/skills/scripts/cluster_health.py
   python3 k8s-troubleshooter/skills/scripts/diagnose_pod.py <namespace> <pod>
   ```

## Design Philosophy

### For SKILL.md Files
- **Workflow-first**: Organize around "how to do X" rather than "what is X"
- **Decision trees**: Help users choose the right approach (see iac-terraform's "Is this reusable?" pattern)
- **Systematic processes**: Provide step-by-step frameworks (see k8s-troubleshooter's 6-phase workflow)
- **Quick references**: Include essential commands for rapid access
- **Link to depth**: Point to references/* for detailed information rather than duplicating

### For Reference Documentation
- **Comprehensive coverage**: Include all relevant details, causes, solutions
- **Searchable structure**: Use clear headings so specific issues can be found quickly
- **Actionable content**: Each issue includes symptoms, causes, diagnostics, remediation, and prevention
- **Real-world focus**: Based on actual production scenarios and common problems

### For Scripts
- **Automation over manual**: Automate repetitive diagnostic tasks
- **Structured output**: Provide clear, parseable results
- **Actionable recommendations**: Don't just report problems, suggest solutions
- **Safe defaults**: Never make destructive changes, only analyze and recommend

## File Conventions

- **Skill names**: lowercase, hyphenated (e.g., `iac-terraform`, `k8s-troubleshooter`)
- **Python scripts**: Descriptive verbs (e.g., `inspect_state.py`, `diagnose_pod.py`, `validate_module.py`)
- **Reference docs**: Noun phrases (e.g., `best_practices.md`, `common_issues.md`, `incident_response.md`)
- **Templates**: UPPERCASE suffix (e.g., `MODULE_TEMPLATE.md`)

## Important Notes

- The `.claude-plugin/marketplace.json` file at the root is the registry that makes skills discoverable
- Each skill's `plugin.json` must match its entry in `marketplace.json`
- SKILL.md files use YAML frontmatter with `name` and `description` fields
- Skills are invoked via `/plugin install` commands after marketplace is added
- Python scripts assume standard DevOps tooling is available (terraform, kubectl, etc.)
