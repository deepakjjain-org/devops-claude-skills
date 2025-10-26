---
name: k8s-troubleshooter
description: Systematic Kubernetes troubleshooting and incident response. Use when diagnosing pod failures, cluster issues, performance problems, networking issues, storage failures, or responding to production incidents. Provides diagnostic workflows, automated health checks, and comprehensive remediation guidance for common Kubernetes problems.
---

# Kubernetes Troubleshooter & Incident Response

Systematic approach to diagnosing and resolving Kubernetes issues in production environments.

## When to Use This Skill

Use this skill when:
- Investigating pod failures (CrashLoopBackOff, ImagePullBackOff, Pending, etc.)
- Responding to production incidents or outages
- Troubleshooting cluster health issues
- Diagnosing networking or service connectivity problems
- Investigating storage/volume issues
- Analyzing performance degradation
- Conducting post-incident analysis

## Core Troubleshooting Workflow

Follow this systematic approach for any Kubernetes issue:

### 1. Gather Context
- What is the observed symptom?
- When did it start?
- What changed recently (deployments, config, infrastructure)?
- What is the scope (single pod, service, node, cluster)?
- What is the business impact (severity level)?

### 2. Initial Triage

Run cluster health check:
```bash
python3 scripts/cluster_health.py
```

This provides an overview of:
- Node health status
- System pod health
- Pending pods
- Failed pods
- Crash loop pods

### 3. Deep Dive Investigation

Based on triage results, focus investigation:

**For Pod Issues:**
```bash
python3 scripts/diagnose_pod.py <namespace> <pod-name>
```

This analyzes:
- Pod phase and readiness
- Container statuses and states
- Restart counts
- Recent events
- Resource usage

**For specific investigations:**
- Review pod details: `kubectl describe pod <pod> -n <namespace>`
- Check logs: `kubectl logs <pod> -n <namespace>`
- Check previous logs if restarting: `kubectl logs <pod> -n <namespace> --previous`
- Check events: `kubectl get events -n <namespace> --sort-by='.lastTimestamp'`

### 4. Identify Root Cause

Consult references/common_issues.md for detailed information on:
- ImagePullBackOff / ErrImagePull
- CrashLoopBackOff
- Pending Pods
- OOMKilled
- Node issues (NotReady, DiskPressure)
- Networking failures
- Storage/PVC issues
- Resource quotas and throttling
- RBAC permission errors

Each issue includes:
- Symptoms
- Common causes
- Diagnostic commands
- Remediation steps
- Prevention strategies

### 5. Apply Remediation

Follow remediation steps from common_issues.md based on root cause identified.

Always:
- Test fixes in non-production first if possible
- Document actions taken
- Monitor for effectiveness
- Have rollback plan ready

### 6. Verify & Monitor

After applying fix:
- Verify issue is resolved
- Monitor for recurrence (15-30 minutes minimum)
- Check related systems
- Update documentation

## Incident Response

For production incidents, follow structured response in references/incident_response.md:

**Severity Assessment:**
- SEV-1 (Critical): Complete outage, data loss, security breach
- SEV-2 (High): Major degradation, significant user impact
- SEV-3 (Medium): Minor impairment, workaround available
- SEV-4 (Low): Cosmetic, minimal impact

**Incident Phases:**
1. **Detection** - Identify and assess
2. **Triage** - Determine scope and impact
3. **Investigation** - Find root cause
4. **Resolution** - Apply fix
5. **Post-Incident** - Document and improve

**Common Incident Scenarios:**
- Complete cluster outage
- Service degradation
- Node failure
- Storage issues
- Security incidents

See references/incident_response.md for detailed playbooks.

## Quick Reference Commands

### Cluster Overview
```bash
kubectl cluster-info
kubectl get nodes
kubectl get pods --all-namespaces | grep -v Running
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -20
```

### Pod Diagnostics
```bash
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --previous
kubectl exec -it <pod> -n <namespace> -- /bin/sh
kubectl get pod <pod> -n <namespace> -o yaml
```

### Node Diagnostics
```bash
kubectl describe node <node>
kubectl top nodes
kubectl top pods --all-namespaces
ssh <node> "systemctl status kubelet"
ssh <node> "journalctl -u kubelet -n 100"
```

### Service & Network
```bash
kubectl describe svc <service> -n <namespace>
kubectl get endpoints <service> -n <namespace>
kubectl get networkpolicies --all-namespaces
```

### Storage
```bash
kubectl get pvc,pv --all-namespaces
kubectl describe pvc <pvc> -n <namespace>
kubectl get storageclass
```

### Resource & Configuration
```bash
kubectl describe resourcequota -n <namespace>
kubectl describe limitrange -n <namespace>
kubectl get rolebindings,clusterrolebindings -n <namespace>
```

## Diagnostic Scripts

### cluster_health.py
Comprehensive cluster health check covering:
- Node status and health
- System pod status (kube-system, etc.)
- Pending pods across all namespaces
- Failed pods
- Pods in crash loops

Usage: `python3 scripts/cluster_health.py`

Best used as first diagnostic step to get overall cluster health snapshot.

### diagnose_pod.py
Detailed pod-level diagnostics:
- Pod phase and status
- Container states (waiting, running, terminated)
- Restart counts and patterns
- Resource configuration issues
- Recent events
- Actionable recommendations

Usage: `python3 scripts/diagnose_pod.py <namespace> <pod-name>`

Best used when investigating specific pod failures or behavior.

## Reference Documentation

### references/common_issues.md
Comprehensive guide to common Kubernetes issues with:
- Detailed symptom descriptions
- Root cause analysis
- Step-by-step diagnostic procedures
- Remediation instructions
- Prevention strategies

Covers:
- Pod issues (ImagePullBackOff, CrashLoopBackOff, Pending, OOMKilled)
- Node issues (NotReady, DiskPressure)
- Networking issues (pod-to-pod communication, service access)
- Storage issues (PVC pending, volume mount failures)
- Resource issues (quota exceeded, CPU throttling)
- Security issues (vulnerabilities, RBAC)

Read this when you identify a specific issue type but need detailed remediation steps.

### references/incident_response.md
Structured incident response framework including:
- Incident response phases (Detection → Triage → Investigation → Resolution → Post-Incident)
- Severity level definitions
- Detailed playbooks for common incident scenarios
- Communication guidelines
- Post-incident review template
- Best practices for prevention, preparedness, response, and recovery

Read this when responding to production incidents or planning incident response procedures.

## Best Practices

**Always:**
- Start with high-level health check before deep diving
- Document symptoms and findings as you investigate
- Check recent changes (deployments, config, infrastructure)
- Preserve logs and state before making destructive changes
- Test fixes in non-production when possible
- Monitor after applying fixes to verify resolution

**Never:**
- Make production changes without understanding impact
- Delete resources without confirming they're safe to remove
- Restart pods repeatedly without investigating root cause
- Apply fixes without documentation
- Skip post-incident review

**Key Principles:**
- Systematic over random troubleshooting
- Evidence-based diagnosis
- Fix root cause, not symptoms
- Learn and improve from each incident
- Prevention is better than reaction
