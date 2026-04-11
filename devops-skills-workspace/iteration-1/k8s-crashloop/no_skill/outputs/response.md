# Troubleshooting: API Pod CrashLoopBackOff with "connection refused" to PostgreSQL

## Situation Summary

After deploying v2.3.1 yesterday, the main API pod is in CrashLoopBackOff. Logs show "connection refused" when attempting to connect to the PostgreSQL service. This means the application container starts, attempts to connect to PostgreSQL, fails, exits with an error, and Kubernetes keeps restarting it with exponential backoff.

---

## Immediate Diagnostic Steps

### Step 1: Confirm the Pod State and Get Details

```bash
# Check the pod status
kubectl get pods -l app=<api-app-label> -o wide

# Describe the pod for events, restart count, and exit codes
kubectl describe pod <api-pod-name>

# Check recent logs (current crash)
kubectl logs <api-pod-name> --tail=100

# Check previous crash logs (if container has restarted)
kubectl logs <api-pod-name> --previous --tail=100
```

Key things to look for in `describe`:
- **Exit code**: Exit code 1 typically means the app crashed due to an unhandled error (like connection failure). Exit code 137 means OOMKilled. Exit code 143 means SIGTERM.
- **Restart count**: Confirms CrashLoopBackOff.
- **Events**: Look for scheduling issues, image pull problems, or readiness/liveness probe failures.

### Step 2: Verify the PostgreSQL Service Is Running

```bash
# Check if the postgres pod is running
kubectl get pods -l app=postgres
# or whatever label selector your postgres uses, e.g.:
kubectl get pods -l app.kubernetes.io/name=postgresql

# Check the postgres service exists and has endpoints
kubectl get svc <postgres-service-name>
kubectl get endpoints <postgres-service-name>

# Check postgres pod logs for any issues
kubectl logs <postgres-pod-name> --tail=100
```

**Critical check**: If `kubectl get endpoints <postgres-service-name>` shows `<none>`, the service has no backing pods -- this is a common cause of "connection refused."

### Step 3: Test Network Connectivity from the API Pod's Namespace

```bash
# Run a temporary debug pod to test connectivity
kubectl run -it --rm debug --image=busybox --restart=Never -- sh

# From inside the debug pod:
nslookup <postgres-service-name>
nslookup <postgres-service-name>.<namespace>.svc.cluster.local
nc -zv <postgres-service-name> 5432
wget -qO- --timeout=5 telnet://<postgres-service-name>:5432 || echo "Connection failed"
```

If DNS resolution works but TCP connection fails, the issue is with the postgres service or pod. If DNS fails, there may be a CoreDNS or service naming issue.

---

## Most Likely Root Causes (Ranked by Probability)

### 1. The v2.3.1 Deployment Changed the Database Connection Configuration

**Why this is the most likely cause**: The timing correlates exactly with the v2.3.1 deployment.

Check:
```bash
# Compare the deployment configuration between versions
kubectl get deployment <api-deployment> -o yaml | grep -A 20 "env:"

# Check if environment variables or ConfigMaps changed
kubectl get configmap <app-config> -o yaml
kubectl get secret <db-secret> -o yaml  # values will be base64 encoded

# Decode a secret value
kubectl get secret <db-secret> -o jsonpath='{.data.DATABASE_URL}' | base64 -d
```

Things to verify:
- **Service name**: Did the postgres service name or namespace change in v2.3.1? (e.g., `postgres` vs `postgresql` vs `postgres-primary`)
- **Port**: Is the app connecting to the correct port (default 5432)?
- **Host format**: Should be `<service-name>.<namespace>.svc.cluster.local` or just `<service-name>` if in the same namespace
- **Database name**: Did the database name change?
- **Credentials**: Were passwords rotated or secrets updated?

**Fix if this is the cause:**
```bash
# Roll back to the previous version immediately to restore service
kubectl rollout undo deployment/<api-deployment>

# Verify the rollback
kubectl rollout status deployment/<api-deployment>
```

Then fix the connection configuration in v2.3.1 and redeploy.

### 2. PostgreSQL Pod/Service Is Down or Unhealthy

```bash
# Check postgres pod status
kubectl get pods -l app=postgres -o wide
kubectl describe pod <postgres-pod>

# Check if postgres is actually accepting connections
kubectl exec -it <postgres-pod> -- pg_isready -U <username>

# Check postgres logs
kubectl logs <postgres-pod> --tail=200
```

Common postgres issues:
- Pod is in CrashLoopBackOff itself (disk full, configuration error)
- Pod is running but postgres process crashed (check logs for FATAL errors)
- Pod was evicted due to resource pressure (check `kubectl describe node`)
- PVC is full or detached

### 3. Service Selector Mismatch

If the v2.3.1 deployment changed labels, or if a postgres redeployment changed labels:

```bash
# Check the service selector
kubectl get svc <postgres-service> -o jsonpath='{.spec.selector}'

# Check the postgres pod labels
kubectl get pods -l app=postgres --show-labels

# Verify they match -- the service selector must match pod labels
```

**Fix:**
```bash
kubectl patch svc <postgres-service> -p '{"spec":{"selector":{"app":"<correct-label>"}}}'
```

### 4. NetworkPolicy Blocking Traffic

If NetworkPolicies were introduced or modified in the v2.3.1 release:

```bash
# List all network policies
kubectl get networkpolicies --all-namespaces

# Check if any policy affects the API or postgres namespace
kubectl describe networkpolicy -n <namespace>
```

**Fix**: Update the NetworkPolicy to allow ingress on port 5432 from the API pods:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-postgres
spec:
  podSelector:
    matchLabels:
      app: postgres
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: <api-app-label>
      ports:
        - port: 5432
          protocol: TCP
```

### 5. PostgreSQL maxconnections Exhausted

If the old API pods are still holding connections or connection pooling changed in v2.3.1:

```bash
# Check current connections on postgres
kubectl exec -it <postgres-pod> -- psql -U <username> -d <database> -c "SELECT count(*) FROM pg_stat_activity;"
kubectl exec -it <postgres-pod> -- psql -U <username> -d <database> -c "SELECT max_conn, used, max_conn - used AS available FROM (SELECT count(*) AS used FROM pg_stat_activity) t1, (SELECT setting::int AS max_conn FROM pg_settings WHERE name = 'max_connections') t2;"
```

**Fix**: Kill idle connections or increase max_connections, or check if a connection pooler (PgBouncer) is configured correctly.

### 6. Cross-Namespace Issue

If v2.3.1 moved the API to a different namespace, or postgres is in a different namespace:

```bash
# Check what namespace the API pod is in
kubectl get pods --all-namespaces | grep <api-pod>

# Check what namespace postgres is in
kubectl get pods --all-namespaces | grep postgres
```

If they are in different namespaces, the connection string must use the fully qualified service name:
```
<service-name>.<namespace>.svc.cluster.local
```

---

## Immediate Remediation: Rollback

If you need to restore service immediately while investigating:

```bash
# Roll back to the previous deployment revision
kubectl rollout undo deployment/<api-deployment>

# Check rollback status
kubectl rollout status deployment/<api-deployment>

# Verify pods are running
kubectl get pods -l app=<api-app-label> -w

# Check rollout history to see revisions
kubectl rollout history deployment/<api-deployment>
```

---

## Systematic Investigation Checklist

Use this checklist to work through the issue methodically:

- [ ] **Pod logs**: Check both current (`kubectl logs`) and previous (`kubectl logs --previous`) container logs
- [ ] **Pod events**: Run `kubectl describe pod` and review the Events section
- [ ] **Postgres running**: Confirm the postgres pod is Running and Ready
- [ ] **Postgres accepting connections**: Run `pg_isready` inside the postgres pod
- [ ] **Service endpoints**: Confirm `kubectl get endpoints <postgres-svc>` is not empty
- [ ] **DNS resolution**: Test DNS from a debug pod (`nslookup <postgres-svc>`)
- [ ] **TCP connectivity**: Test port 5432 from a debug pod (`nc -zv <postgres-svc> 5432`)
- [ ] **Connection string**: Verify host, port, database name, username, password in the deployment env/configmap/secret
- [ ] **Diff v2.3.1 vs previous**: Compare deployment manifests for any config changes
- [ ] **NetworkPolicies**: Check if any policy blocks traffic between API and postgres
- [ ] **Resource limits**: Check if postgres or API pods are being OOMKilled or throttled
- [ ] **Node placement**: Verify both pods are scheduled and nodes are healthy (`kubectl get nodes`)

---

## Preventing Future Occurrences

1. **Add a startup probe or init container** that waits for postgres to be ready before the main container starts:
   ```yaml
   initContainers:
     - name: wait-for-postgres
       image: busybox:1.36
       command: ['sh', '-c', 'until nc -zv $DB_HOST 5432; do echo "Waiting for postgres..."; sleep 2; done']
       env:
         - name: DB_HOST
           valueFrom:
             configMapKeyRef:
               name: app-config
               key: db-host
   ```

2. **Implement readiness probes** that check database connectivity so the pod is not marked Ready until it can actually serve traffic.

3. **Use connection retry logic** in the application -- most ORMs and database libraries support configurable retry with backoff (e.g., SQLAlchemy's `pool_pre_ping`, Django's `CONN_HEALTH_CHECKS`).

4. **Add deployment smoke tests** in your CI/CD pipeline that validate database connectivity in a staging environment before promoting to production.

5. **Use `kubectl diff`** before applying changes to see exactly what will change:
   ```bash
   kubectl diff -f deployment.yaml
   ```

6. **Implement progressive rollouts** (canary or blue-green) so that a broken deployment does not take down 100% of traffic immediately.

---

## Quick Reference: Key Commands

| Action | Command |
|--------|---------|
| Check pod status | `kubectl get pods -l app=<label> -o wide` |
| View crash logs | `kubectl logs <pod> --previous --tail=100` |
| Describe pod | `kubectl describe pod <pod>` |
| Check service endpoints | `kubectl get endpoints <svc>` |
| Test connectivity | `kubectl run debug --rm -it --image=busybox -- nc -zv <svc> 5432` |
| Check postgres health | `kubectl exec <pg-pod> -- pg_isready` |
| Rollback deployment | `kubectl rollout undo deployment/<name>` |
| View rollout history | `kubectl rollout history deployment/<name>` |
| Compare configs | `kubectl get deployment <name> -o yaml` |
| Check network policies | `kubectl get networkpolicies` |
