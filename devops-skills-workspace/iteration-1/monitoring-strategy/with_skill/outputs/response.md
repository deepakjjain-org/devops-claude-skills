# Complete Monitoring Strategy for 12 Microservices on Kubernetes

## Budget: $3,000/month | Timeline: Launch-ready in 4 weeks

---

## Executive Summary

For 12 microservices on Kubernetes with a $3k/month budget, I recommend the **self-hosted Grafana LGTM stack**: Prometheus + Grafana + Loki + Tempo. This gives you full metrics, logs, and traces coverage within budget, with no vendor lock-in. Based on the tool comparison data, this stack runs approximately $1,500/month for infrastructure at 100-host scale, leaving you headroom for growth and storage. A managed SaaS like Datadog would cost $8,000+/month for this footprint -- well outside your budget.

---

## 1. Tool Selection and Cost Breakdown

### Recommended Stack: Prometheus + Grafana + Loki + Tempo (LGTM)

| Component | Purpose | Monthly Cost Estimate |
|-----------|---------|---------------------|
| **Prometheus** (+ kube-prometheus-stack) | Metrics collection, alerting rules | $0 (open source) |
| **Grafana** | Dashboards, visualization, alerting UI | $0 (open source) |
| **Grafana Loki** | Log aggregation | $0 (open source) |
| **Grafana Tempo** | Distributed tracing | $0 (open source) |
| **Alertmanager** | Alert routing and notification | $0 (bundled with Prometheus) |
| **OpenTelemetry Collector** | Unified telemetry pipeline | $0 (open source) |
| **Infrastructure** (storage, compute for monitoring) | Run the stack on K8s | ~$800-1,200/month |
| **PagerDuty (or Opsgenie)** | On-call incident management | ~$200-400/month |
| **Object storage** (S3/GCS for Loki + Tempo) | Long-term retention | ~$100-200/month |
| **Total** | | **~$1,200-1,800/month** |

This leaves $1,200-1,800/month of budget headroom for growth, additional storage, or upgrading to Grafana Cloud later if ops burden becomes a concern.

### Why Not Other Options?

- **Datadog**: $8,000+/month for 12 services -- nearly 3x your budget
- **ELK Stack**: $4,000+/month and requires a dedicated ops team -- too expensive and operationally heavy
- **CloudWatch**: $2,000/month but limited querying, no tracing, and locks you into AWS
- **Grafana Cloud**: $3,000/month -- viable if you want zero ops, but uses your entire budget with no headroom. Consider this as a Phase 2 migration if self-hosting becomes too burdensome
- **New Relic**: Free tier is generous but costs escalate quickly past 100GB/month with 12 services

---

## 2. Metrics Strategy: The Four Golden Signals

Every one of your 12 microservices must expose these four signals. This is non-negotiable -- it is the foundation everything else builds on.

### Per-Service Metrics (RED Method for request-driven services)

**Rate** -- requests per second:
```promql
sum(rate(http_requests_total{service="$service"}[5m]))
```

**Errors** -- error rate percentage:
```promql
sum(rate(http_requests_total{service="$service",status=~"5.."}[5m]))
  /
sum(rate(http_requests_total{service="$service"}[5m])) * 100
```

**Duration** -- p95 latency:
```promql
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket{service="$service"}[5m])) by (le)
)
```

### Infrastructure Metrics (USE Method)

**Utilization**:
```promql
# CPU per node
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory per node
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

**Saturation**:
```promql
# Disk I/O saturation
rate(node_disk_io_time_seconds_total[5m]) * 100
```

### Instrumentation Requirements

Every service must expose a `/metrics` endpoint using a Prometheus client library:

- **Python**: `prometheus_client` (Counter, Histogram, Gauge)
- **Node.js**: `prom-client`
- **Go**: `prometheus/client_golang`
- **Java**: Micrometer with Prometheus registry

Minimum metrics each service must export:
```
http_requests_total{method, endpoint, status}        # Counter
http_request_duration_seconds{method, endpoint}       # Histogram (with buckets)
http_requests_in_progress                              # Gauge
```

### Cardinality Rules (Critical for Prometheus Health)

Labels must be low-cardinality. Never use user IDs, email addresses, IP addresses, or random IDs as label values.

Good: `service (12) x environment (3) x method (5) x status (5) = 900 time series`
Bad: `service (12) x user_id (100K) = 1.2M time series` -- this will kill Prometheus

### Collection Intervals

- Application metrics: 15-30s scrape interval
- Infrastructure metrics (node-exporter): 30s
- kube-state-metrics: 30s

---

## 3. SLOs and Error Budgets

### Recommended Initial SLOs (Start Loose, Tighten Over Time)

| Service Tier | Availability SLO | Latency SLO (p95) | Error Rate SLO | Error Budget/Month |
|-------------|-------------------|--------------------|-----------------|--------------------|
| **Tier 1** (user-facing, critical path) | 99.9% | < 500ms | < 0.1% | 43.2 minutes downtime |
| **Tier 2** (supporting services) | 99.5% | < 1s | < 0.5% | 3.6 hours downtime |
| **Tier 3** (internal/batch) | 99% | < 5s | < 1% | 7.2 hours downtime |

Classify your 12 services into these tiers. Typically: 2-3 Tier 1 services (API gateway, auth, core business logic), 5-6 Tier 2, and 3-4 Tier 3.

### Error Budget Policy

Define consequences at different budget consumption levels:

- **Error budget > 50% remaining**: Deploy freely, experiment, take calculated risks
- **Error budget 20-50% remaining**: Increase testing, review recent changes, deploy once per day max
- **Error budget < 20% remaining**: Freeze non-critical deploys, focus on reliability
- **Error budget exhausted**: Complete deploy freeze (except rollbacks), all hands on reliability, executive escalation

### SLO Monitoring with Prometheus Recording Rules

```yaml
groups:
  - name: sli_recording
    interval: 30s
    rules:
      # Per-service request success rate
      - record: sli:request_success:ratio
        expr: |
          sum(rate(http_requests_total{status!~"5.."}[5m])) by (service)
          /
          sum(rate(http_requests_total[5m])) by (service)

      # Per-service p95 latency
      - record: sli:request_latency:p95
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
          )

      # Error budget burn rate (1h window)
      - record: slo:error_budget_burn_rate:1h
        expr: |
          (1 - sli:request_success:ratio) / 0.001
```

### Automation

Use the SLO calculator script to validate targets and compute burn rates:
```bash
# Show reference table of SLO vs downtime
python3 monitoring-observability/scripts/slo_calculator.py --table

# Calculate for a specific service
python3 monitoring-observability/scripts/slo_calculator.py availability \
  --slo 99.9 \
  --total-requests 1000000 \
  --failed-requests 500 \
  --period-days 30
```

---

## 4. Alerting Strategy

### Alert Severity Routing

| Severity | Response Time | Channel | Examples |
|----------|--------------|---------|----------|
| **Critical** | Page immediately (< 5 min ack) | PagerDuty -> on-call engineer | Service down, SLO violation, error budget fast burn |
| **Warning** | Ticket, review in business hours | Slack #alerts-warning | Elevated error rate, resource warning, slow burn |
| **Info** | Log for awareness, no action | Slack #alerts-info | Deployment completed, scaling event |

### Multi-Window Burn Rate Alerting (Primary Alert Pattern)

This is Google's recommended SLO alerting approach. It catches both fast and slow degradation.

**Fast burn** (1h window, critical -- budget exhausted in ~2 days):
```yaml
- alert: ErrorBudgetFastBurn
  expr: |
    (
      sum(rate(http_requests_total{status=~"5.."}[1h])) by (service)
      /
      sum(rate(http_requests_total[1h])) by (service)
    ) > (14.4 * 0.001)  # 14.4x burn rate for 99.9% SLO
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Fast error budget burn - {{ $labels.service }}"
    runbook_url: "https://runbooks.internal/error-budget-burn"
```

**Slow burn** (6h window, warning -- budget exhausted in ~5 days):
```yaml
- alert: ErrorBudgetSlowBurn
  expr: |
    (
      sum(rate(http_requests_total{status=~"5.."}[6h])) by (service)
      /
      sum(rate(http_requests_total[6h])) by (service)
    ) > (6 * 0.001)  # 6x burn rate for 99.9% SLO
  for: 30m
  labels:
    severity: warning
```

### Kubernetes-Specific Alerts (Deploy Immediately)

Use the provided Kubernetes alert template (`monitoring-observability/assets/templates/prometheus-alerts/kubernetes-alerts.yml`) which covers:

- **Pod alerts**: CrashLooping, NotReady, OOMKilled
- **Deployment alerts**: Replica mismatch, rollout stuck
- **Node alerts**: NodeNotReady (critical), MemoryPressure, DiskPressure, HighCPU
- **Resource alerts**: CPU throttling, memory approaching limits
- **PersistentVolume alerts**: Filling up (warning at 85%), critically full (critical at 95%)
- **Job alerts**: Failed jobs, CronJobs not running

### Web Application Alerts (Per Service)

Use the provided webapp alert template (`monitoring-observability/assets/templates/prometheus-alerts/webapp-alerts.yml`) which covers:

- Error budget burn (fast and slow)
- Service down detection
- Latency degradation (p95 and p99)
- CPU and memory warnings
- Traffic anomalies (spikes and drops)
- Database connection pool exhaustion
- External API error rates

### Alertmanager Configuration

```yaml
route:
  group_by: ['alertname', 'service', 'namespace']
  group_wait: 10s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'default-slack'

  routes:
    # Critical -> PagerDuty
    - match:
        severity: critical
      receiver: pagerduty
      group_wait: 10s
      repeat_interval: 5m

    # Warning -> Slack channel
    - match:
        severity: warning
      receiver: slack-warnings
      group_wait: 30s
      repeat_interval: 12h

    # Info -> Slack (low priority)
    - match:
        severity: info
      receiver: slack-info
      repeat_interval: 24h

inhibit_rules:
  # If service is down, suppress latency alerts for that service
  - source_match:
      alertname: WebAppDown
    target_match:
      alertname: HighLatencyP95
    equal: ['service']

  # If node is down, suppress all pod alerts on that node
  - source_match:
      alertname: NodeNotReady
    target_match_re:
      alertname: '(PodCrashLooping|ContainerCPUThrottling|ContainerMemoryUsageHigh)'
    equal: ['node']
```

### Alert Quality Validation

Before deploying alert rules, validate them:
```bash
python3 monitoring-observability/scripts/alert_quality_checker.py /path/to/alert-rules/
```

This checks: naming conventions, required labels/annotations, PromQL validity, presence of `for` clauses to prevent flapping, and runbook URLs.

---

## 5. Log Aggregation with Loki

### Architecture

```
Pods (stdout/stderr) -> Promtail (DaemonSet) -> Loki -> Grafana
```

### Structured Logging Standard (Mandatory for All 12 Services)

Every service must log in JSON format to stdout. Every log entry must include:

```json
{
  "timestamp": "2026-04-09T14:32:15.123Z",
  "level": "error",
  "message": "Payment processing failed",
  "service": "payment-service",
  "version": "1.4.2",
  "environment": "production",
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "trace_id": "0af7651916cd43dd8448eb211c80319c",
  "span_id": "b7ad6b7169203331",
  "error_type": "GatewayTimeout",
  "duration_ms": 5000
}
```

Required fields: `timestamp`, `level`, `message`, `service`, `environment`.
Strongly recommended: `request_id`, `trace_id`, `span_id` (for correlation with traces), `duration_ms`.

### Implementation by Language

- **Python**: Use `structlog` with JSON renderer
- **Node.js**: Use `winston` with JSON format
- **Go**: Use `zap` with production JSON config
- **Java**: Use Logback with `logstash-logback-encoder`

### Log Levels Policy

- **DEBUG**: Disabled in production (enable via config flag for troubleshooting)
- **INFO**: Business events (order placed, user login), state changes
- **WARN**: Slow operations, retries, approaching resource limits
- **ERROR**: Failed requests, caught exceptions, integration failures
- **CRITICAL**: Service cannot function (database connection lost, fatal configuration error)

### Loki Label Design

Keep labels low-cardinality for Loki performance:

```yaml
# Good labels (low cardinality)
labels:
  namespace: "production"
  service: "payment-service"
  level: "error"

# DO NOT use as labels (high cardinality)
# user_id, request_id, trace_id -- these go in the log line, not labels
```

### Log Retention Strategy

- **Hot** (Loki): 14 days -- full query performance
- **Cold** (S3/GCS): 90 days -- query via restore if needed
- **Archive**: 1 year for compliance (compressed in object storage)

### PII Redaction

Implement PII redaction before logging (not after). Redact email addresses, credit card numbers, SSNs, and any other PII at the application level.

---

## 6. Distributed Tracing with Tempo and OpenTelemetry

### Why Tracing Matters for 12 Microservices

With 12 services, a single user request may traverse 4-8 services. Without tracing, debugging latency issues or failures across service boundaries is nearly impossible.

### Architecture

```
Services (OTel SDK) -> OTel Collector (DaemonSet) -> Grafana Tempo -> Grafana
```

### OpenTelemetry Collector Configuration

Deploy the OTel Collector as a DaemonSet in Kubernetes. Use the provided template at `monitoring-observability/assets/templates/otel-config/collector-config.yaml` as a starting point. For your stack, configure these pipelines:

```yaml
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, tail_sampling, resource]
      exporters: [otlp/tempo]

    metrics:
      receivers: [otlp, prometheus, hostmetrics, k8s_cluster]
      processors: [memory_limiter, batch, resource]
      exporters: [prometheus]

    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch, resource]
      exporters: [loki]
```

### Tail Sampling Strategy (Critical for Cost Control)

In production with 12 services, you cannot keep 100% of traces. Use tail sampling in the OTel Collector:

```yaml
processors:
  tail_sampling:
    decision_wait: 10s
    num_traces: 100
    policies:
      # Always keep errors
      - name: error-policy
        type: status_code
        status_code:
          status_codes: [ERROR]

      # Always keep slow traces
      - name: latency-policy
        type: latency
        latency:
          threshold_ms: 1000

      # Sample 5% of everything else
      - name: probabilistic-policy
        type: probabilistic
        probabilistic:
          sampling_percentage: 5
```

This ensures you always capture errors and slow requests (the ones you actually need to debug) while keeping storage costs manageable.

### Auto-Instrumentation

Use auto-instrumentation wherever possible to minimize developer effort:

- **Python**: `FlaskInstrumentor`, `RequestsInstrumentor`, `SQLAlchemyInstrumentor`
- **Node.js**: `HttpInstrumentation`, `ExpressInstrumentation`, `MongoDBInstrumentation`
- **Go**: `otelhttp`, `otelsql`
- **Java**: OpenTelemetry Java Agent (zero-code instrumentation)

### Context Propagation

All services must propagate W3C Trace Context headers (`traceparent`, `tracestate`). This happens automatically with OTel auto-instrumentation. For any manual HTTP calls, inject and extract context:

```python
from opentelemetry.propagate import inject, extract

# Outgoing requests: inject
headers = {}
inject(headers)
requests.get("https://other-service/api", headers=headers)

# Incoming requests: extract
ctx = extract(request.headers)
```

### Log-Trace Correlation

Include `trace_id` and `span_id` in every log entry. In Grafana, configure Loki derived fields to link directly to Tempo traces -- clicking a trace ID in a log line opens the trace waterfall view.

---

## 7. Dashboard Strategy

### Dashboard Hierarchy

Build dashboards in layers:

1. **Executive Overview** (1 dashboard): Overall system health for all 12 services at a glance
2. **Per-Service Dashboards** (12 dashboards): Detailed RED metrics for each service
3. **Kubernetes Platform Dashboard** (1 dashboard): Cluster health, node status, resource usage
4. **SLO Dashboard** (1 dashboard): SLO compliance, error budgets, burn rates for all services

### Executive Overview Layout

```
Row 1: Single stats (total RPS, overall error rate, worst p95 latency, unhealthy services count)
Row 2: Request rate per service (stacked graph)
Row 3: Error rate per service (heatmap or multi-line)
Row 4: p95 latency per service
Row 5: Resource utilization (CPU/memory aggregates)
```

### Per-Service Dashboard Layout

Follow the standard top-down pattern (8-12 panels max):

```
Row 1: Health single stats [RPS] [Error%] [P95 Latency] [P99 Latency]
Row 2: Request rate + Error rate (dual graph)
Row 3: Latency distribution (p50, p95, p99 overlaid)
Row 4: Resource usage (CPU, Memory)
Row 5: Dependencies (DB query time, external API latency)
```

### Automated Dashboard Generation

Generate per-service dashboards using the provided script:
```bash
# For each web-facing service
python3 monitoring-observability/scripts/dashboard_generator.py webapp \
  --title "Payment Service" \
  --service payment-service \
  --output dashboards/payment-service.json

# For Kubernetes overview
python3 monitoring-observability/scripts/dashboard_generator.py kubernetes \
  --title "Production Cluster" \
  --namespace production \
  --output dashboards/k8s-production.json
```

### Template Variables

Every dashboard should use Grafana template variables for filtering:
- `$namespace` (kubernetes namespace)
- `$service` (service name)
- `$environment` (prod/staging)
- `$pod` (individual pod)

---

## 8. Health Check Endpoints

Every service must expose two health endpoints:

### Liveness Probe: `/healthz`
Returns 200 if the process is alive. Used by Kubernetes to decide whether to restart the pod.

### Readiness Probe: `/ready`
Returns 200 only if the service can handle traffic (database connected, dependencies reachable). Used by Kubernetes to decide whether to route traffic to the pod.

### Health Check Requirements
- Return JSON with `status` field
- Include version and build info
- Include dependency checks (for readiness)
- Response time under 1 second
- No caching headers

### Validation

Validate your health check endpoints:
```bash
python3 monitoring-observability/scripts/health_check_validator.py \
  https://service.internal/healthz \
  https://service.internal/ready \
  --verbose
```

---

## 9. On-Call and Incident Response

### On-Call Rotation

- **Primary on-call**: First responder (5-minute ack SLA)
- **Secondary on-call**: Escalation backup (auto-escalate after 15 minutes)
- **Rotation length**: 1 week per engineer
- **Handoff**: Monday morning with a review of open incidents and upcoming deploys

### Runbook Requirements

Every critical and warning alert must have a linked runbook. Use the provided template at `monitoring-observability/assets/templates/runbooks/incident-runbook-template.md` as the standard format. Each runbook must include:

1. What the alert means and user impact
2. Step-by-step investigation commands (copy-pasteable)
3. Common scenarios and solutions
4. Immediate actions (< 5 min), short-term (< 30 min), long-term (post-incident)
5. Escalation path with contacts

### Alert Quality Metrics to Track

- **Mean Time to Acknowledge (MTTA)**: Target < 5 minutes
- **Mean Time to Resolve (MTTR)**: Target < 30 minutes
- **False Positive Rate**: Target < 10%
- **Alert Coverage**: Target > 80% of incidents preceded by an alert

---

## 10. Implementation Roadmap

### Week 1: Foundation

- [ ] Deploy kube-prometheus-stack (Prometheus + Grafana + Alertmanager + node-exporter + kube-state-metrics) via Helm
- [ ] Deploy Grafana Loki + Promtail
- [ ] Deploy Grafana Tempo
- [ ] Deploy OpenTelemetry Collector as DaemonSet
- [ ] Set up PagerDuty/Opsgenie integration
- [ ] Deploy Kubernetes alert rules from template

### Week 2: Application Instrumentation

- [ ] Define structured logging standard and distribute to all service teams
- [ ] Add Prometheus metrics endpoints to all 12 services (RED metrics minimum)
- [ ] Add OpenTelemetry auto-instrumentation to all 12 services
- [ ] Ensure all services log JSON to stdout with required fields
- [ ] Add health check endpoints (`/healthz`, `/ready`) to all services

### Week 3: Dashboards, Alerts, and SLOs

- [ ] Generate per-service Grafana dashboards (use `dashboard_generator.py`)
- [ ] Build executive overview dashboard
- [ ] Build SLO compliance dashboard
- [ ] Deploy per-service alert rules (burn rate alerts, latency, error rate)
- [ ] Configure Alertmanager routing (critical -> PagerDuty, warning -> Slack)
- [ ] Configure log-trace correlation in Grafana (Loki derived fields -> Tempo)

### Week 4: Validation and Hardening

- [ ] Run alert quality checker on all alert rules
- [ ] Validate health check endpoints with `health_check_validator.py`
- [ ] Write runbooks for all critical alerts
- [ ] Set up on-call rotation in PagerDuty
- [ ] Test alert routing end-to-end (fire test alerts, verify PagerDuty and Slack delivery)
- [ ] Conduct a fire drill: simulate a service failure and verify detection, alerting, and response
- [ ] Classify all 12 services into Tier 1/2/3 and document SLO targets
- [ ] Document error budget policy and get team sign-off

---

## 11. Available Automation Scripts

The monitoring-observability skill includes these scripts for ongoing operations:

| Script | Purpose | When to Use |
|--------|---------|-------------|
| `analyze_metrics.py` | Detect anomalies and trends in Prometheus metrics | Weekly metric reviews, investigating performance issues |
| `alert_quality_checker.py` | Audit alert rules against best practices | Before deploying new alerts, quarterly alert review |
| `slo_calculator.py` | Calculate SLO compliance, error budgets, burn rates | Monthly SLO reviews |
| `log_analyzer.py` | Parse logs for errors, patterns, stack traces | Incident investigation |
| `dashboard_generator.py` | Generate Grafana dashboards from templates | Onboarding new services |
| `health_check_validator.py` | Validate health check endpoints | Pre-launch validation, CI/CD pipeline |

---

## 12. Key Reference Documents

For deeper guidance on any topic, consult these references within the monitoring-observability skill:

- **metrics_design.md**: Four Golden Signals, RED/USE methods, metric types, cardinality management, naming conventions
- **alerting_best_practices.md**: Alert design patterns, multi-window burn rate, routing, inhibition, runbook structure, on-call practices
- **logging_guide.md**: Structured logging implementation, log aggregation patterns (Loki, ELK, CloudWatch), PII redaction, sampling
- **tracing_guide.md**: OpenTelemetry instrumentation (Python, Node.js, Go, Java), context propagation, sampling strategies, backend comparison
- **slo_sla_guide.md**: Choosing SLIs, setting SLO targets, error budget policies, burn rate alerting, monthly reporting
- **tool_comparison.md**: Full comparison of Prometheus, Datadog, ELK, Loki, CloudWatch, Jaeger, Tempo, and other tools

---

## Summary of Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Metrics** | Prometheus (self-hosted) | Industry standard for K8s, free, massive ecosystem |
| **Visualization** | Grafana | Best-in-class dashboards, unified view of metrics/logs/traces |
| **Logs** | Grafana Loki | Cost-effective ($100-500/mo vs $3K+ for ELK), Prometheus-like label model |
| **Traces** | Grafana Tempo | Cost-effective (object storage backend), native Grafana integration |
| **Telemetry Pipeline** | OpenTelemetry Collector | Vendor-neutral, unified pipeline for all signals |
| **Alerting** | Multi-window burn rate | Catches both fast and slow degradation, ties to SLOs |
| **Incident Management** | PagerDuty or Opsgenie | Proven escalation, on-call rotation, integrates with Alertmanager |
| **Default SLO** | 99.9% availability | Standard for production services, 43 min/month error budget |
| **Trace Sampling** | Tail sampling (errors: 100%, slow: 100%, rest: 5%) | Captures what matters, controls storage cost |
| **Log Retention** | 14 days hot, 90 days cold | Balances query speed with storage cost |
