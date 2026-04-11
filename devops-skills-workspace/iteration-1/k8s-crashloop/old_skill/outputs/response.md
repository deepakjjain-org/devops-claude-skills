# CrashLoopBackOff Diagnosis: API Pod "connection refused" to PostgreSQL

## Incident Assessment

**Symptom:** Main API pod in CrashLoopBackOff after v2.3.1 deployment
**Error:** "connection refused" to postgres service
**Trigger:** Deployment of v2.3.1 yesterday
**Likely Severity:** SEV-2 (High) -- Major functionality degraded, significant user impact since the main API is down

Following the k8s-troubleshooter systematic workflow: Gather Context -> Initial Triage -> Deep Dive -> Root Cause -> Remediation -> Verify.

---

## Phase 1: Gather Context

Key questions already answered:
- **Observed symptom:** CrashLoopBackOff with "connection refused" to postgres
- **When it started:** After v2.3.1 deployment yesterday
- **What changed:** New application version deployed
- **Scope:** Main API pod (service-level impact)

---

## Phase 2: Initial Triage

Run the cluster health check to get an overall picture:

```bash
python3 scripts/cluster_health.py
```

This will show whether this is isolated to your API pod or part of a broader cluster issue (node problems, other crashlooping pods, etc.).

Then run a namespace-level health check for a fuller picture:

```bash
python3 scripts/check_namespace.py <your-namespace>
```

This will show pod status, service health and endpoints, deployment availability, and recent events -- all of which are relevant to diagnosing a service connectivity failure.

---

## Phase 3: Deep Dive Investigation

Run pod-level diagnostics on the crashing API pod:

```bash
python3 scripts/diagnose_pod.py <namespace> <api-pod-name>
```

Then run these commands manually in sequence:

### 3a. Confirm the CrashLoopBackOff and get details

```bash
# Get the pod name
kubectl get pods -n <namespace> -l app=<your-api-label>

# Describe the pod for events and state details
kubectl describe pod <api-pod-name> -n <namespace>

# Check current logs
kubectl logs <api-pod-name> -n <namespace>

# Check previous container's logs (the crashed instance)
kubectl logs <api-pod-name> -n <namespace> --previous
```

Look for the exact "connection refused" error -- note the hostname/IP and port it is trying to reach.

### 3b. Verify the PostgreSQL service is healthy

```bash
# Check if the postgres pod is running
kubectl get pods -n <namespace> -l app=postgres

# Check postgres service exists and has the right configuration
kubectl get svc postgres -n <namespace>
kubectl describe svc postgres -n <namespace>

# CRITICAL: Check if the service has endpoints
kubectl get endpoints postgres -n <namespace>
```

**If endpoints are empty**, that means no healthy postgres pods are backing the service. This is a very common cause of "connection refused."

### 3c. Check postgres pod health

```bash
# Describe the postgres pod
kubectl describe pod <postgres-pod-name> -n <namespace>

# Check postgres logs
kubectl logs <postgres-pod-name> -n <namespace>

# Check if postgres is actually listening
kubectl exec -it <postgres-pod-name> -n <namespace> -- pg_isready
```

### 3d. Test connectivity from the API pod's perspective

```bash
# Exec into a running pod (or use a debug container) and test the connection
kubectl run tmp-debug --rm -i --tty --image=busybox -n <namespace> -- sh

# Inside the debug pod:
nslookup postgres.<namespace>.svc.cluster.local
nc -zv postgres.<namespace>.svc.cluster.local 5432
```

---

## Phase 4: Identify Root Cause

Based on the CrashLoopBackOff reference (common_issues.md), the "connection refused" to postgres after a new deployment points to one of these likely root causes, ordered by probability:

### Most Likely: Changes in v2.3.1 broke the database connection

1. **Database connection string changed in v2.3.1** -- The new version may reference a different service name, port, or use a different environment variable for the connection string.
   - Check: Compare the environment variables/configmaps between v2.3.0 and v2.3.1
   ```bash
   kubectl rollout history deployment/<api-deployment> -n <namespace>
   kubectl get deployment <api-deployment> -n <namespace> -o yaml
   ```
   - Look at `DB_HOST`, `DATABASE_URL`, `POSTGRES_HOST`, or whatever env var your app uses for the postgres connection.

2. **ConfigMap or Secret was updated alongside the deployment** -- Credentials or connection parameters may have changed.
   ```bash
   kubectl get configmap -n <namespace>
   kubectl get secrets -n <namespace>
   # Compare with what the pod is mounting/referencing
   ```

3. **PostgreSQL service name or port mismatch** -- The API may be trying to connect to a service name or port that does not match the actual postgres service.
   ```bash
   kubectl get svc -n <namespace> | grep postgres
   ```

### Also Possible: PostgreSQL itself is down

4. **Postgres pod crashed or is not ready** -- If postgres was restarted alongside the API deployment, it may not have come up yet or may be in a crash loop itself.
   ```bash
   kubectl get pods -n <namespace> -l app=postgres
   ```

5. **Network policy blocking traffic** -- A new network policy may have been applied that blocks the API pod from reaching postgres.
   ```bash
   kubectl get networkpolicies -n <namespace>
   ```

6. **Postgres is in a different namespace** and the service reference does not include the namespace qualifier (e.g., `postgres` vs `postgres.database-namespace.svc.cluster.local`).

7. **Liveness probe too aggressive** -- If the API has a liveness probe that checks before the database connection can be established, Kubernetes may be killing the pod before it fully starts. The v2.3.1 deployment may have changed the startup timing.
   ```bash
   kubectl get deployment <api-deployment> -n <namespace> -o jsonpath='{.spec.template.spec.containers[0].livenessProbe}'
   ```

---

## Phase 5: Remediation

### Immediate Fix: Rollback if this is blocking production

If the API is the main production service and it is down, **rollback first, investigate second**:

```bash
kubectl rollout undo deployment/<api-deployment> -n <namespace>

# Verify the rollback
kubectl rollout status deployment/<api-deployment> -n <namespace>
kubectl get pods -n <namespace> -l app=<your-api-label>
```

This gets your service back up while you investigate the v2.3.1 changes.

### Root Cause Fixes (based on what the investigation reveals):

**If the connection string/env vars changed:**
```bash
# Fix the configmap or deployment env vars
kubectl edit configmap <api-config> -n <namespace>
# or
kubectl set env deployment/<api-deployment> DB_HOST=postgres.<namespace>.svc.cluster.local -n <namespace>
```

**If postgres service has no endpoints:**
```bash
# Check and fix the service selector
kubectl describe svc postgres -n <namespace>
# Ensure the selector labels match the postgres pod labels
kubectl get pods -n <namespace> --show-labels | grep postgres
```

**If postgres itself is down:**
```bash
# Restart the postgres pod/deployment
kubectl rollout restart deployment/postgres -n <namespace>
# or for a statefulset
kubectl rollout restart statefulset/postgres -n <namespace>
```

**If a network policy is blocking traffic:**
```bash
# Inspect network policies
kubectl describe networkpolicy -n <namespace>
# Update or remove the offending policy
```

**If the liveness probe is killing the pod too early:**
```bash
# Increase initialDelaySeconds on the liveness probe
kubectl patch deployment <api-deployment> -n <namespace> --type json -p '[
  {"op": "replace", "path": "/spec/template/spec/containers/0/livenessProbe/initialDelaySeconds", "value": 60}
]'
```

### Recommended: Add init container for dependency checking

To prevent this class of issue in the future, add an init container that waits for postgres to be ready before the API starts:

```yaml
initContainers:
- name: wait-for-postgres
  image: busybox:1.36
  command: ['sh', '-c', 'until nc -zv postgres 5432; do echo "waiting for postgres..."; sleep 2; done']
```

---

## Phase 6: Verify and Monitor

After applying the fix:

```bash
# Verify the API pod is running and stable
kubectl get pods -n <namespace> -l app=<your-api-label> -w

# Check that it stays running (wait at least 5 minutes to confirm no further crashes)
kubectl logs -f <api-pod-name> -n <namespace>

# Verify the API can connect to postgres
kubectl logs <api-pod-name> -n <namespace> | grep -i "database\|postgres\|connected"

# Verify the service endpoints are populated
kubectl get endpoints <api-service-name> -n <namespace>

# Run the namespace health check again to confirm everything is clean
python3 scripts/check_namespace.py <namespace>
```

Monitor for at least 15-30 minutes to ensure no recurrence.

---

## Prevention Recommendations

Based on the k8s-troubleshooter best practices:

1. **Use init containers** for dependency checks so pods do not crash-loop when dependencies are temporarily unavailable.
2. **Configure startup probes** (separate from liveness probes) with generous timeouts for applications that need time to establish database connections.
3. **Add retry logic with backoff** in the application code for database connections rather than failing immediately on "connection refused."
4. **Test deployments in staging** with the exact same service dependencies before promoting to production.
5. **Use readiness probes** so the pod is not added to service endpoints until it has a confirmed database connection.
6. **Pin database connection configuration** in a versioned ConfigMap so changes are explicit and reviewable.
7. **Implement rollout strategies** (e.g., `maxUnavailable: 0`) so the old pods are not terminated until new pods are confirmed ready.

---

## Summary of Diagnostic Commands to Run Now

```bash
# 1. Cluster-level health check
python3 scripts/cluster_health.py

# 2. Namespace-level health check
python3 scripts/check_namespace.py <namespace>

# 3. Pod-level diagnostics
python3 scripts/diagnose_pod.py <namespace> <api-pod-name>

# 4. Get the crash logs
kubectl logs <api-pod-name> -n <namespace> --previous

# 5. Check postgres service endpoints (key diagnostic)
kubectl get endpoints postgres -n <namespace>

# 6. Check postgres pod health
kubectl get pods -n <namespace> -l app=postgres

# 7. Test DNS and connectivity
kubectl run tmp-debug --rm -i --tty --image=busybox -n <namespace> -- nslookup postgres

# 8. Compare deployment changes
kubectl rollout history deployment/<api-deployment> -n <namespace>
```

Start with the logs (step 4) and the endpoints check (step 5) -- these two commands will most likely reveal the root cause immediately.
