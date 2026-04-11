# Complete Monitoring Strategy for 12 Microservices on Kubernetes

## Budget: $3,000/month | Platform: Kubernetes | Timeline: Launch in 1 month

---

## Executive Summary

Given your Kubernetes environment and $3,000/month budget, I recommend the **Grafana LGTM Stack** (Loki, Grafana, Tempo, Mimir/Prometheus). This is the optimal choice because:

- It fits within your budget (self-hosted ~$1,500/month, leaving headroom for growth)
- It is the industry standard for Kubernetes monitoring
- It provides full observability: metrics, logs, traces, and dashboards in a unified platform
- It avoids vendor lock-in while remaining production-grade

Datadog would cost ~$8,000/month for 12 services and is out of budget. CloudWatch would work but lacks the depth needed for microservices debugging. The Grafana stack gives you the best cost-to-capability ratio.

> **Reference**: Tool comparison details from `references/tool_comparison.md` -- Prometheus + Loki + Tempo comes in at ~$1,500/month for self-hosted, well within your $3k budget even with buffer.

---

## Phase 1: Foundation (Week 1-2) -- Metrics & Infrastructure Monitoring

### 1.1 Deploy the Prometheus Stack via kube-prometheus-stack

Install the kube-prometheus-stack Helm chart, which bundles Prometheus, Grafana, Alertmanager, and node-exporter:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=15d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi \
  --set grafana.persistence.enabled=true \
  --set grafana.persistence.size=10Gi
```

This immediately gives you:
- Node-level metrics (CPU, memory, disk, network) for all Kubernetes nodes
- Kubernetes metrics (pods, deployments, services, jobs) via kube-state-metrics
- Grafana with pre-built Kubernetes dashboards
- Alertmanager for alert routing

### 1.2 Instrument All 12 Microservices with the Four Golden Signals

Every one of your 12 microservices must expose these metrics. This is non-negotiable -- it is the foundation of your entire monitoring strategy.

> **Reference**: The Four Golden Signals framework from `references/metrics_design.md`

**For each request-driven service, use the RED Method:**

| Signal | Metric Name | Type | Description |
|--------|-------------|------|-------------|
| **Rate** | `http_requests_total` | Counter | Total requests, labeled by method, endpoint, status |
| **Errors** | `http_requests_total{status=~"5.."}` | Counter | 5xx errors (subset of above) |
| **Duration** | `http_request_duration_seconds` | Histogram | Request latency distribution |
| **Saturation** | `process_cpu_seconds_total`, `process_resident_memory_bytes` | Gauge | Resource utilization |

**Python (Flask/FastAPI) instrumentation example:**
```python
from prometheus_client import Counter, Histogram, Gauge

# Rate + Errors (single counter with status label)
requests_total = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

# Duration
request_duration = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration',
    ['method', 'endpoint'],
    buckets=[0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
)

# Saturation
requests_in_progress = Gauge(
    'http_requests_in_progress',
    'HTTP requests currently being processed'
)
```

**Node.js (Express) instrumentation example:**
```javascript
const client = require('prom-client');

const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
});

const httpRequestsTotal = new client.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status_code']
});
```

**Go instrumentation example:**
```go
import "github.com/prometheus/client_golang/prometheus"

var httpRequestsTotal = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "http_requests_total",
        Help: "Total HTTP requests",
    },
    []string{"method", "endpoint", "status"},
)

var httpRequestDuration = prometheus.NewHistogramVec(
    prometheus.HistogramOpts{
        Name:    "http_request_duration_seconds",
        Help:    "HTTP request duration",
        Buckets: prometheus.DefBuckets,
    },
    []string{"method", "endpoint"},
)
```

### 1.3 Cardinality Rules (Critical for 12 Services)

With 12 microservices, cardinality can explode fast. Follow these rules strictly:

**ALLOWED labels (low cardinality):**
- `service` (12 values)
- `environment` (prod, staging = 2 values)
- `method` (GET, POST, PUT, DELETE = ~5 values)
- `status` (200, 201, 400, 404, 500 = ~10 values)
- `endpoint` (keep to route patterns, not parameterized URLs)

**NEVER use as labels (high cardinality):**
- User IDs
- Request IDs
- IP addresses
- Timestamps
- Order/transaction IDs

**Cardinality calculation:**
```
12 services x 2 envs x 5 methods x 10 statuses x 20 endpoints = 24,000 time series (safe)
12 services x 2 envs x 5 methods x 10 statuses x 100,000 user_ids = 120M time series (DISASTER)
```

> **Reference**: Cardinality best practices from `references/metrics_design.md`

### 1.4 ServiceMonitor Configuration

Create ServiceMonitors so Prometheus auto-discovers your services:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: microservices
  namespace: monitoring
  labels:
    release: monitoring
spec:
  namespaceSelector:
    matchNames:
      - production
  selector:
    matchLabels:
      monitoring: "true"
  endpoints:
    - port: metrics
      interval: 15s
      path: /metrics
```

Then label each service:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
  labels:
    monitoring: "true"
spec:
  ports:
    - name: metrics
      port: 9090
```

### 1.5 Validate Cluster Health

After deploying, use the cluster health script to verify everything is working:

```bash
python3 scripts/cluster_health.py
```

Use the metrics analyzer to check that data is flowing:

```bash
python3 scripts/analyze_metrics.py prometheus \
  --endpoint http://prometheus.monitoring:9090 \
  --query 'up{job=~".*"}' \
  --hours 1
```

> **Script**: `scripts/analyze_metrics.py` -- Detect anomalies and trends in your Prometheus metrics

---

## Phase 2: Logging (Week 2-3) -- Structured Logs with Loki

### 2.1 Deploy Grafana Loki

Loki is the cost-effective choice for your budget. Unlike ELK ($4,000/month), Loki only indexes labels, not full text, which dramatically reduces storage costs.

```bash
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=50Gi \
  --set promtail.enabled=true
```

Promtail runs as a DaemonSet and automatically collects all container logs from every pod.

### 2.2 Structured Logging Standard (Mandatory for All 12 Services)

Every microservice MUST output structured JSON logs. Unstructured logs are not acceptable for a microservices architecture.

> **Reference**: Structured logging patterns from `references/logging_guide.md`

**Required fields in every log entry:**

```json
{
  "timestamp": "2026-04-09T14:32:15.123Z",
  "level": "error",
  "message": "Payment processing failed",
  "service": "payment-service",
  "trace_id": "550e8400-e29b-41d4-a716-446655440000",
  "span_id": "a2fb4a1d1a96d312",
  "request_id": "req-789",
  "environment": "production",
  "error_type": "GatewayTimeout",
  "duration_ms": 5000
}
```

**Key rules:**
1. Always include `trace_id` -- this links logs to distributed traces
2. Always include `service` name -- essential for filtering across 12 services
3. Use ISO 8601 timestamps
4. Use consistent log levels: DEBUG, INFO, WARN, ERROR, FATAL
5. Redact PII before logging (emails, SSNs, etc.)
6. In production, set minimum log level to INFO (never DEBUG)

### 2.3 Log Levels Policy

| Level | When to Use | Example |
|-------|-------------|---------|
| **DEBUG** | Development only | Function entry/exit, variable values |
| **INFO** | Business events | "Order placed", "User logged in", "Payment succeeded" |
| **WARN** | Degraded but functional | Slow query, retry attempt, cache miss |
| **ERROR** | Failed operation | Payment failed, external API timeout, validation error |
| **FATAL** | Service cannot continue | Database connection lost, critical config missing |

### 2.4 Loki Query Patterns for Microservices

```logql
# All errors across all services in the last hour
{namespace="production"} |= "error" | json | level="error"

# Errors for a specific service
{namespace="production", app="order-service"} | json | level="error"

# Find all logs for a specific trace
{namespace="production"} | json | trace_id="550e8400-e29b-41d4-a716-446655440000"

# Slow requests (>1s)
{namespace="production"} | json | duration_ms > 1000

# Error rate by service (last 5 min)
sum by (service) (count_over_time({namespace="production"} | json | level="error" [5m]))
```

### 2.5 Log Retention Policy

| Tier | Retention | Purpose |
|------|-----------|---------|
| Hot (Loki) | 15 days | Active querying and debugging |
| Cold (S3/GCS) | 90 days | Compliance and deep investigation |

This keeps storage costs under control. Configure Loki's `retention_period: 360h` (15 days).

---

## Phase 3: Distributed Tracing (Week 3) -- OpenTelemetry + Tempo

### 3.1 Why Tracing is Critical for 12 Microservices

With 12 services, a single user request may flow through 4-8 services. Without tracing, debugging "why is this request slow?" becomes a nightmare of log correlation. Tracing gives you a waterfall view of every hop.

> **Reference**: OpenTelemetry instrumentation guide from `references/tracing_guide.md`

### 3.2 Deploy Grafana Tempo

```bash
helm install tempo grafana/tempo \
  --namespace monitoring \
  --set persistence.enabled=true \
  --set persistence.size=20Gi
```

### 3.3 Deploy the OpenTelemetry Collector

The OTel Collector receives traces from all services, processes them, and exports to Tempo.

> **Template**: Production-ready OTel Collector config from `assets/templates/otel-config/collector-config.yaml`

Deploy as a DaemonSet for sidecar-free collection:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: otel-collector
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: otel-collector
  template:
    spec:
      containers:
        - name: collector
          image: otel/opentelemetry-collector-contrib:latest
          ports:
            - containerPort: 4317  # gRPC OTLP
            - containerPort: 4318  # HTTP OTLP
          volumeMounts:
            - name: config
              mountPath: /etc/otelcol
      volumes:
        - name: config
          configMap:
            name: otel-collector-config
```

Key collector configuration for your setup:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 10s
    send_batch_size: 1024
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
  tail_sampling:
    decision_wait: 10s
    policies:
      - name: error-policy
        type: status_code
        status_code:
          status_codes: [ERROR]
      - name: latency-policy
        type: latency
        latency:
          threshold_ms: 1000
      - name: probabilistic-policy
        type: probabilistic
        probabilistic:
          sampling_percentage: 10

exporters:
  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, tail_sampling, batch]
      exporters: [otlp/tempo]
```

### 3.4 Instrument Services with OpenTelemetry

Each microservice should use auto-instrumentation where possible:

**Python:**
```bash
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap -a install
```

```python
# Set via environment variables (12-factor app style)
OTEL_SERVICE_NAME=order-service
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
OTEL_TRACES_SAMPLER=parentbased_always_on
```

**Node.js:**
```bash
npm install @opentelemetry/sdk-node @opentelemetry/auto-instrumentations-node
```

**Go:**
```bash
go get go.opentelemetry.io/otel
go get go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp
```

### 3.5 Sampling Strategy

For 12 microservices in production, use **tail-based sampling** through the OTel Collector:

| Rule | Sampling Rate | Rationale |
|------|---------------|-----------|
| All errors | 100% | Always capture failures for debugging |
| Slow requests (>1s) | 100% | Always capture performance issues |
| Normal requests | 10% | Sufficient for trend analysis, saves storage |

This keeps trace storage manageable while ensuring you never miss an error or slow request.

### 3.6 Context Propagation

All 12 services MUST propagate trace context. Use W3C Trace Context (the standard):

```
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
```

Ensure every inter-service call (HTTP, gRPC, message queue) propagates this header. Most OTel auto-instrumentation handles this automatically for HTTP clients and frameworks.

---

## Phase 4: Alerting Strategy (Week 3-4) -- SLO-Based Alerts

### 4.1 Define SLOs for Each Service Tier

Not all 12 services are equal. Categorize them by criticality:

> **Reference**: SLO target guidance from `references/slo_sla_guide.md`

| Tier | Services | Availability SLO | Latency SLO (p95) | Error Budget/Month |
|------|----------|-------------------|--------------------|--------------------|
| **Tier 1 (Critical)** | API Gateway, Auth, Payment | 99.95% | < 300ms | 21.6 minutes |
| **Tier 2 (Important)** | Order, User, Inventory | 99.9% | < 500ms | 43.2 minutes |
| **Tier 3 (Standard)** | Notifications, Analytics, Recommendations | 99.5% | < 1s | 3.6 hours |

**Use the SLO calculator to validate targets:**
```bash
python3 scripts/slo_calculator.py --table

python3 scripts/slo_calculator.py availability \
  --slo 99.9 \
  --total-requests 1000000 \
  --failed-requests 1500 \
  --period-days 30
```

> **Script**: `scripts/slo_calculator.py` -- Calculate SLO compliance, error budgets, and burn rates

### 4.2 Alert Rules -- Multi-Window Burn Rate

Use SLO-based burn rate alerting rather than static thresholds. This prevents alert fatigue while catching real issues.

> **Reference**: Multi-window burn rate alerting from `references/alerting_best_practices.md`

**Apply the provided webapp alert templates directly:**

> **Template**: `assets/templates/prometheus-alerts/webapp-alerts.yml` -- Production-ready SLO burn rate alerts
> **Template**: `assets/templates/prometheus-alerts/kubernetes-alerts.yml` -- Kubernetes platform alerts

**Critical alerts (page immediately):**

```yaml
groups:
  - name: slo_burn_rate
    rules:
      # Fast burn: exhausts monthly budget in 2 days
      - alert: ErrorBudgetFastBurn
        expr: |
          (
            sum by (service) (rate(http_requests_total{status=~"5.."}[1h]))
            /
            sum by (service) (rate(http_requests_total[1h]))
          ) > (14.4 * 0.001)
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Fast error budget burn on {{ $labels.service }}"
          description: "Error budget will be exhausted in 2 days at current rate."
          runbook_url: "https://runbooks.internal/error-budget-burn"

      # Service completely down
      - alert: ServiceDown
        expr: absent(up{job=~".*-service"})
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.job }} is not reporting metrics"
          runbook_url: "https://runbooks.internal/service-down"
```

**Warning alerts (create ticket, review in hours):**

```yaml
      # Slow burn: exhausts monthly budget in 5 days
      - alert: ErrorBudgetSlowBurn
        expr: |
          (
            sum by (service) (rate(http_requests_total{status=~"5.."}[6h]))
            /
            sum by (service) (rate(http_requests_total[6h]))
          ) > (6 * 0.001)
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "Elevated error budget burn on {{ $labels.service }}"

      # High p95 latency
      - alert: HighLatencyP95
        expr: |
          histogram_quantile(0.95,
            sum by (le, service) (rate(http_request_duration_seconds_bucket[5m]))
          ) > 0.5
        for: 10m
        labels:
          severity: warning
```

### 4.3 Kubernetes Platform Alerts

In addition to application alerts, deploy these Kubernetes alerts for all 12 services:

| Alert | Severity | Condition |
|-------|----------|-----------|
| PodCrashLooping | Warning | Restarts > 0 in 15m for 5m |
| PodOOMKilled | Warning | OOMKilled termination detected |
| DeploymentReplicasMismatch | Warning | Desired != available for 15m |
| NodeNotReady | Critical | Node not ready for 5m |
| NodeMemoryPressure | Warning | Memory pressure for 5m |
| ContainerCPUThrottling | Warning | Throttling > 0.5 for 10m |
| PersistentVolumeFillingUp | Warning | < 15% free for 10m |

**Audit your alert rules after writing them:**
```bash
python3 scripts/alert_quality_checker.py /path/to/alert-rules/
```

> **Script**: `scripts/alert_quality_checker.py` -- Checks naming conventions, required labels/annotations, PromQL quality, and for clauses

### 4.4 Alert Routing with Alertmanager

```yaml
# alertmanager.yml
route:
  group_by: ['alertname', 'service', 'namespace']
  group_wait: 10s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'default-slack'

  routes:
    # Critical: page on-call engineer
    - match:
        severity: critical
      receiver: pagerduty-critical
      group_wait: 10s
      repeat_interval: 5m

    # Warning: send to Slack
    - match:
        severity: warning
      receiver: team-slack
      group_wait: 30s
      repeat_interval: 12h

inhibit_rules:
  # If service is down, suppress latency alerts for that service
  - source_match:
      alertname: ServiceDown
    target_match_re:
      alertname: '(HighLatency.*|ErrorBudget.*)'
    equal: ['service']

  # If node is down, suppress pod alerts on that node
  - source_match:
      alertname: NodeNotReady
    target_match_re:
      alertname: '(PodCrashLooping|PodOOMKilled|ContainerCPUThrottling)'
    equal: ['node']
```

### 4.5 Runbooks

Every alert MUST have a runbook. Use the provided template:

> **Template**: `assets/templates/runbooks/incident-runbook-template.md`

Each runbook must include:
1. **Context**: What this alert means, what is the user impact
2. **Investigation steps**: Specific commands and dashboard links
3. **Common causes**: Top 3-5 reasons this fires
4. **Resolution steps**: Immediate (< 5 min), short-term (< 30 min), long-term
5. **Escalation path**: Who to contact if unresolved after 30 minutes

---

## Phase 5: Dashboards (Week 4) -- Grafana Visualization

### 5.1 Dashboard Hierarchy

Create a 3-tier dashboard structure:

```
Level 1: Platform Overview (1 dashboard)
    Shows: All 12 services at a glance, overall SLO compliance
    Audience: Engineering leadership, on-call engineer

Level 2: Service Dashboards (12 dashboards, 1 per service)
    Shows: RED metrics, dependencies, resource usage for one service
    Audience: Service team, on-call investigating an alert

Level 3: Infrastructure (2-3 dashboards)
    Shows: Kubernetes cluster health, node resources, storage
    Audience: Platform team
```

### 5.2 Generate Dashboards Using the Dashboard Generator

```bash
# Generate a dashboard for each service
for service in api-gateway auth-service payment-service order-service \
  user-service inventory-service notification-service analytics-service \
  recommendation-service search-service catalog-service shipping-service; do

  python3 scripts/dashboard_generator.py webapp \
    --title "${service} Dashboard" \
    --service ${service} \
    --output dashboards/${service}.json
done

# Generate the Kubernetes infrastructure dashboard
python3 scripts/dashboard_generator.py kubernetes \
  --title "K8s Production Cluster" \
  --namespace production \
  --output dashboards/k8s-production.json
```

> **Script**: `scripts/dashboard_generator.py` -- Generates Grafana dashboards with proper RED metric panels

### 5.3 Platform Overview Dashboard Layout

```
+-----------------------------------------------------------+
|  Platform Health                                           |
|  [Total RPS: 15k] [Overall Error%: 0.02%] [P95: 180ms]   |
+-----------------------------------------------------------+
+-----------------------------------------------------------+
|  Service Status Grid (12 services)                        |
|  [api-gw: OK] [auth: OK] [payment: OK] [order: WARN]     |
|  [user: OK] [inventory: OK] [notif: OK] [analytics: OK]  |
|  [reco: OK] [search: OK] [catalog: OK] [shipping: OK]    |
+-----------------------------------------------------------+
+-----------------------------------------------------------+
|  Error Budget Status (by tier)                            |
|  Tier 1: 87% remaining | Tier 2: 92% remaining           |
+-----------------------------------------------------------+
+-----------------------------------------------------------+
|  Request Rate by Service (stacked graph)                  |
+-----------------------------------------------------------+
+-----------------------------------------------------------+
|  P95 Latency by Service (line graph)                      |
+-----------------------------------------------------------+
```

### 5.4 Per-Service Dashboard Layout

Follow the standard layout from the skill:

```
+-------------------------------------------+
|  Health Stats                              |
|  [RPS] [Error%] [P95 Latency] [In-Flight] |
+-------------------------------------------+
+-------------------------------------------+
|  Request Rate & Error Rate (graphs)        |
+-------------------------------------------+
+-------------------------------------------+
|  Latency Distribution: p50, p95, p99       |
+-------------------------------------------+
+-------------------------------------------+
|  CPU & Memory Usage                        |
+-------------------------------------------+
+-------------------------------------------+
|  Dependency Health (DB, external APIs)     |
+-------------------------------------------+
```

Use template variables for filtering: `$environment`, `$service`, `$pod`.

---

## Phase 6: Health Checks (Pre-Launch)

### 6.1 Health Check Endpoints

Every microservice must expose two health check endpoints:

**Liveness** (`/healthz`): Is the process alive?
```json
{"status": "ok", "timestamp": "2026-04-09T14:32:15Z"}
```

**Readiness** (`/ready`): Can it serve traffic?
```json
{
  "status": "ok",
  "checks": {
    "database": "ok",
    "cache": "ok",
    "upstream-auth": "ok"
  },
  "version": "1.4.2",
  "timestamp": "2026-04-09T14:32:15Z"
}
```

### 6.2 Validate Health Checks Before Launch

```bash
python3 scripts/health_check_validator.py \
  https://api-gateway.prod/healthz \
  https://auth-service.prod/healthz \
  https://payment-service.prod/healthz \
  --verbose
```

> **Script**: `scripts/health_check_validator.py` -- Validates response code, response time, JSON format, status field, dependency checks, and cache headers

---

## Budget Breakdown

| Component | Monthly Cost | Notes |
|-----------|-------------|-------|
| **Prometheus** (metrics) | $0 (open source) | 3-node HA on existing K8s capacity |
| **Grafana** (dashboards) | $0 (open source) | Single instance with persistent storage |
| **Loki** (logs) | $0 (open source) + ~$200 storage | 50GB persistent volume |
| **Tempo** (traces) | $0 (open source) + ~$100 storage | 20GB persistent volume, 10% sampling |
| **Kubernetes resources** | ~$800-1,200 | 3-4 additional nodes for monitoring stack |
| **PagerDuty** (on-call) | ~$400 | Professional plan for 5 engineers |
| **Object storage** (S3/GCS for long-term) | ~$100-200 | Log and metric archival |
| **Buffer for growth** | ~$800-1,100 | Headroom for the first 3 months |
| **Total** | **~$1,700-2,100** | Well within $3,000 budget |

This leaves $900-1,300/month of budget headroom for:
- Scaling as traffic grows
- Adding synthetic monitoring later
- Upgrading to Grafana Cloud if ops burden becomes too high

---

## Implementation Timeline

| Week | Milestone | Deliverables |
|------|-----------|-------------|
| **Week 1** | Prometheus + Grafana deployed | Cluster metrics flowing, K8s dashboards live |
| **Week 2** | All 12 services instrumented | RED metrics from every service, ServiceMonitors configured |
| **Week 2-3** | Loki + Promtail deployed | Structured logs from all services, log queries working |
| **Week 3** | Tempo + OTel Collector deployed | Distributed traces flowing, sampling configured |
| **Week 3-4** | Alerts + SLOs configured | Burn rate alerts, Alertmanager routing, PagerDuty integration |
| **Week 4** | Dashboards + runbooks | Platform overview, 12 service dashboards, runbooks for all critical alerts |
| **Pre-launch** | Health check validation | All endpoints validated, load test with monitoring |

---

## Post-Launch: Ongoing Operations

### Monthly SLO Review

Hold a monthly SLO review meeting. Use this data:

```bash
# Calculate SLO compliance for each service
python3 scripts/slo_calculator.py availability \
  --slo 99.9 \
  --total-requests 1000000 \
  --failed-requests 800 \
  --period-days 30
```

> **Reference**: Monthly SLO report template from `references/slo_sla_guide.md`

Review:
1. Did each service meet its SLO?
2. How much error budget was consumed?
3. Which incidents consumed the most budget?
4. Should any SLO targets be adjusted?

### Quarterly Tuning

- Review alert noise: Are any alerts firing too frequently without action? Remove or adjust them.
- Review cardinality: Check for metric explosion. Remove unused labels.
- Review storage: Ensure retention policies are working. Archive old data.
- Review sampling: Adjust trace sampling rates based on actual volume.

### Key PromQL Queries to Know

```promql
# Overall request rate across all services
sum(rate(http_requests_total[5m])) by (service)

# Overall error rate
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
  /
sum(rate(http_requests_total[5m])) by (service) * 100

# P95 latency per service
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
)

# Error budget burn rate (for 99.9% SLO)
(
  sum by (service) (rate(http_requests_total{status=~"5.."}[1h]))
  /
  sum by (service) (rate(http_requests_total[1h]))
) / 0.001

# Top 5 slowest endpoints
topk(5,
  histogram_quantile(0.95,
    sum(rate(http_request_duration_seconds_bucket[5m])) by (le, endpoint)
  )
)
```

---

## Summary of Scripts and References Used

### Scripts (from the monitoring-observability skill)
| Script | Purpose | When to Use |
|--------|---------|-------------|
| `scripts/analyze_metrics.py` | Detect anomalies in Prometheus metrics | After initial deployment, ongoing analysis |
| `scripts/alert_quality_checker.py` | Audit alert rules against best practices | After writing alert rules |
| `scripts/slo_calculator.py` | Calculate SLO compliance and error budgets | Monthly SLO reviews |
| `scripts/dashboard_generator.py` | Generate Grafana dashboards | During dashboard setup |
| `scripts/health_check_validator.py` | Validate health endpoints | Pre-launch validation |
| `scripts/log_analyzer.py` | Parse logs for errors and patterns | Incident investigation |

### Reference Documents
| Reference | Key Content |
|-----------|-------------|
| `references/metrics_design.md` | Four Golden Signals, RED/USE methods, cardinality rules |
| `references/alerting_best_practices.md` | Multi-window burn rate, alert routing, runbook structure |
| `references/slo_sla_guide.md` | SLO targets, error budgets, burn rates, monthly reporting |
| `references/tracing_guide.md` | OpenTelemetry instrumentation, sampling strategies |
| `references/logging_guide.md` | Structured logging, log levels, aggregation patterns |
| `references/tool_comparison.md` | Cost comparison confirming Grafana stack as optimal choice |

### Templates
| Template | Purpose |
|----------|---------|
| `assets/templates/prometheus-alerts/webapp-alerts.yml` | Production-ready SLO burn rate alerts |
| `assets/templates/prometheus-alerts/kubernetes-alerts.yml` | Kubernetes platform alerts |
| `assets/templates/otel-config/collector-config.yaml` | OpenTelemetry Collector configuration |
| `assets/templates/runbooks/incident-runbook-template.md` | Incident runbook structure |
