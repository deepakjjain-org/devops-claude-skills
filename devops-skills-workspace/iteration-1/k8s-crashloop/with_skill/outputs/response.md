# Troubleshooting: API Pod CrashLoopBackOff -- "connection refused" to PostgreSQL

## 1. Gather Context (Skill Workflow -- Phase 1)

Based on what you have reported:

- **Symptom:** Main API pod in CrashLoopBackOff
- **When it started:** Yesterday, after deploying v2.3.1
- **What changed:** New deployment (v2.3.1)
- **Scope:** Single service (main API pod), but with downstream dependency failure (PostgreSQL)
- **Severity Assessment:** This is likely **SEV-2 (High)** -- major functionality is degraded, your main API is down and restarting in a loop, which likely affects a large subset of users.

The "connection refused" error to your PostgreSQL service tells us the application is starting but immediately failing when it tries to establish a database connection. This is consistent with CrashLoopBackOff cause #5 from the skill's common issues reference: **"Missing dependencies (DB, cache, etc.)"**.

---

## 2. Initial Triage (Skill Workflow -- Phase 2)

Run the cluster-wide health check first to understand the broader picture:

```bash
python3 /path/to/k8s-troubleshooter/skills/scripts/cluster_health.py
```

This will show you:
- Whether nodes are healthy
- Whether system pods are running
- How many pods are in CrashLoopBackOff across the cluster
- Whether there are other pending or failed pods (which could indicate a wider issue)

---

## 3. Deep Dive Investigation (Skill Workflow -- Phase 3)

### Step 3a: Diagnose the API Pod

Run the pod diagnostics script:

```bash
python3 /path/to/k8s-troubleshooter/skills/scripts/diagnose_pod.py <namespace> <api-pod-name>
```

This will analyze the pod phase, container states, restart counts, recent events, and provide actionable recommendations.

### Step 3b: Manual Diagnostic Commands

Run these commands in sequence to build a complete picture:

```bash
# Get the pod name (it may have a hash suffix)
kubectl get pods -n <namespace> -l app=<your-api-label>

# Check current pod status and events
kubectl describe pod <api-pod-name> -n <namespace>

# Check current container logs (the crash output)
kubectl logs <api-pod-name> -n <namespace>

# Check PREVIOUS container logs (the last crash -- often more informative)
kubectl logs <api-pod-name> -n <namespace> --previous

# Check namespace events sorted by time
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

### Step 3c: Investigate the PostgreSQL Service

Since the error is "connection refused," the PostgreSQL service itself is the key area to investigate:

```bash
# Check if the PostgreSQL pod is running
kubectl get pods -n <namespace> -l app=postgres
# (adjust the label selector to match your postgres deployment)

# Check the PostgreSQL service exists and has endpoints
kubectl get svc -n <namespace> | grep postgres
kubectl get endpoints <postgres-service-name> -n <namespace>

# Describe the PostgreSQL service
kubectl describe svc <postgres-service-name> -n <namespace>

# Check PostgreSQL pod logs
kubectl logs <postgres-pod-name> -n <namespace>

# Test connectivity from within a debug pod
kubectl run tmp-debug --rm -i --tty --image=busybox -n <namespace> -- sh
# Then inside the pod:
#   nc -zv <postgres-service-name> 5432
#   nslookup <postgres-service-name>.<namespace>.svc.cluster.local
```

---

## 4. Root Cause Analysis (Skill Workflow -- Phase 4)

Given the symptoms ("connection refused" after a v2.3.1 deployment), here are the most likely root causes, ordered by probability:

### Cause A: v2.3.1 Changed the Database Connection Configuration

**Most likely if:** The PostgreSQL pod itself is healthy and accepting connections from other services.

Check:
```bash
# Compare the environment variables or configmaps between the old and new deployment
kubectl get deployment <api-deployment> -n <namespace> -o yaml | grep -A 20 env

# Check if a ConfigMap or Secret was changed
kubectl get configmap -n <namespace>
kubectl get secret -n <namespace>

# Check rollout history to see what changed
kubectl rollout history deployment/<api-deployment> -n <namespace>
```

Common issues in this scenario:
- Database hostname changed (e.g., from `postgres` to `postgresql`, or a new service name)
- Port changed (e.g., hardcoded 5432 but new config uses a different port)
- New environment variable names in v2.3.1 that were not set in the deployment manifest
- Connection string format changed in the new version
- TLS/SSL requirements added in v2.3.1 but the PostgreSQL service does not support it

### Cause B: PostgreSQL Service Is Down or Unreachable

**Most likely if:** The PostgreSQL pod is not running, has no endpoints, or is in a bad state.

Check:
```bash
# Are there endpoints for the postgres service?
kubectl get endpoints <postgres-service-name> -n <namespace>
# If "ENDPOINTS" column shows "<none>", the service has no backing pods

# Is postgres actually running?
kubectl get pods -l app=postgres -n <namespace>

# Check postgres logs for errors
kubectl logs <postgres-pod> -n <namespace>
```

If endpoints are empty:
- The PostgreSQL pod may have crashed or been evicted
- The service selector labels may not match the PostgreSQL pod labels
- A recent change may have affected the PostgreSQL deployment

### Cause C: Network Policy Blocking Traffic

**Most likely if:** PostgreSQL is running and healthy, but the API pod still cannot reach it.

Check:
```bash
# List network policies in the namespace
kubectl get networkpolicies -n <namespace>

# Describe any network policies
kubectl describe networkpolicy -n <namespace>
```

If network policies exist, verify they allow traffic from the API pod to the PostgreSQL pod on port 5432. The v2.3.1 deployment may have changed pod labels, causing the pod to no longer match an allow rule.

### Cause D: PostgreSQL Is in a Different Namespace

**Most likely if:** The service DNS name needs a fully qualified domain.

Check if the API pod is using the correct service address:
- Same namespace: `postgres-service` or `postgres-service:5432`
- Cross-namespace: `postgres-service.<other-namespace>.svc.cluster.local:5432`

### Cause E: Init Container or Startup Probe Issue

**Most likely if:** v2.3.1 added new init containers or changed startup probes.

```bash
kubectl get pod <api-pod-name> -n <namespace> -o yaml | grep -A 30 initContainers
kubectl get pod <api-pod-name> -n <namespace> -o yaml | grep -A 10 startupProbe
```

---

## 5. Remediation Steps (Skill Workflow -- Phase 5)

Based on the root cause you identify, apply the appropriate fix:

### If Cause A (Config changed in v2.3.1):

**Immediate fix -- Roll back to the previous working version:**
```bash
kubectl rollout undo deployment/<api-deployment> -n <namespace>
```

Then investigate the configuration difference and fix the v2.3.1 manifests before redeploying.

**Targeted fix -- correct the configuration:**
```bash
# If environment variables are wrong, patch the deployment
kubectl set env deployment/<api-deployment> -n <namespace> \
  DATABASE_HOST=<correct-postgres-service-name> \
  DATABASE_PORT=5432

# Or edit the deployment directly
kubectl edit deployment/<api-deployment> -n <namespace>
```

### If Cause B (PostgreSQL is down):

```bash
# Restart the PostgreSQL deployment
kubectl rollout restart deployment/<postgres-deployment> -n <namespace>

# Or if it is a StatefulSet
kubectl rollout restart statefulset/<postgres-statefulset> -n <namespace>

# Check if PVC is bound (storage issue could prevent postgres from starting)
kubectl get pvc -n <namespace>
```

If PostgreSQL cannot start, check its logs and events for the specific failure reason (disk full, OOMKilled, image issue, etc.).

### If Cause C (Network policy blocking):

```bash
# Temporarily, you can delete the network policy to confirm it is the issue
# (only do this in non-production, or if you understand the security implications)
kubectl delete networkpolicy <policy-name> -n <namespace>

# Proper fix: Update the network policy to allow traffic from the API pods
# Ensure the pod selector matches the v2.3.1 pod labels
```

### If Cause D (Wrong namespace/DNS):

Update the database hostname in the API deployment configuration to use the fully qualified service name:
```
<postgres-service>.<namespace>.svc.cluster.local
```

### General: Add Init Container for Dependency Check (Prevention)

To prevent this class of failure in the future, add an init container that waits for PostgreSQL to be available before the main container starts:

```yaml
initContainers:
- name: wait-for-postgres
  image: busybox:1.36
  command: ['sh', '-c', 'until nc -z <postgres-service> 5432; do echo "Waiting for PostgreSQL..."; sleep 2; done']
```

This converts a CrashLoopBackOff into a graceful wait, giving you time to fix the dependency without the API pod burning through restart backoff timers.

---

## 6. Verify and Monitor (Skill Workflow -- Phase 6)

After applying your fix:

```bash
# Watch the pod status
kubectl get pods -n <namespace> -w

# Verify the API pod is Running and Ready
kubectl get pods -n <namespace> -l app=<your-api-label>

# Check the logs to confirm successful database connection
kubectl logs <new-api-pod-name> -n <namespace>

# Test the API endpoint
kubectl exec -it <api-pod> -n <namespace> -- curl -s http://localhost:<port>/health
# Or from outside:
curl -s https://<your-api-endpoint>/health

# Monitor for 15-30 minutes for recurrence
kubectl get events -n <namespace> --sort-by='.lastTimestamp' --watch
```

Run the diagnostic script again to confirm clean health:
```bash
python3 /path/to/k8s-troubleshooter/skills/scripts/diagnose_pod.py <namespace> <new-api-pod-name>
```

---

## 7. Quick Decision Tree

```
"connection refused" to PostgreSQL
|
+-- Is the PostgreSQL pod running?
|   |
|   +-- NO --> Fix PostgreSQL first (check logs, PVC, resources)
|   |
|   +-- YES --> Does the PostgreSQL service have endpoints?
|       |
|       +-- NO --> Service selector labels do not match pod labels. Fix the service or pod labels.
|       |
|       +-- YES --> Can you connect from a debug pod? (nc -zv <svc> 5432)
|           |
|           +-- NO --> Network policy or DNS issue. Check NetworkPolicies, check DNS resolution.
|           |
|           +-- YES --> The database IS reachable. The issue is in the API pod's connection config.
|                       Check env vars, secrets, configmaps for the correct host/port/credentials.
|                       Compare with the previous working deployment revision.
```

---

## 8. Prevention Recommendations

Based on the skill's best practices for avoiding CrashLoopBackOff:

1. **Use init containers for dependency checks** -- Wait for PostgreSQL to be ready before starting the API container.
2. **Implement proper readiness probes** -- The API should only be marked ready after it successfully connects to the database.
3. **Add retry logic with backoff in the application** -- The application should retry database connections on startup rather than crashing immediately.
4. **Use ConfigMaps/Secrets for database configuration** -- Avoid hardcoding connection strings in the image. This makes it easy to audit and change without rebuilding.
5. **Test database connectivity in your CI/CD pipeline** -- Validate that the connection configuration is correct before deploying.
6. **Set appropriate liveness probe `initialDelaySeconds`** -- Give the application enough time to establish database connections before the liveness probe starts checking.
7. **Maintain rollback capability** -- Always be ready to `kubectl rollout undo` to the last known working version.

---

## Summary

Your API pod is crash-looping because it cannot connect to PostgreSQL after the v2.3.1 deployment. The most probable cause is a configuration change in v2.3.1 (database host, port, credentials, or connection string). Start by checking whether PostgreSQL itself is healthy and has service endpoints, then compare the database connection configuration between the current and previous deployment revision. If you need to restore service immediately, roll back with `kubectl rollout undo` while you investigate the v2.3.1 configuration.
