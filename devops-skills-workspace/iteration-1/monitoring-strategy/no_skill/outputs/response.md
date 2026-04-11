# Complete Monitoring Strategy for 12 Kubernetes Microservices

## Budget: $3,000/month | Timeline: Production-ready within 1 month

---

## Executive Summary

This strategy uses an open-source-first stack deployed on your existing Kubernetes cluster to maximize value within the $3K/month budget. The core stack (Prometheus, Grafana, Loki, Tempo) costs $0 in licensing -- your spend goes entirely toward infrastructure resources (compute, storage) and one modest managed service for on-call alerting.

---

## 1. Recommended Stack and Budget Allocation

### Tool Selection

| Layer | Tool | Why | Cost |
|-------|------|-----|------|
| **Metrics** | Prometheus + Thanos | Industry standard for K8s, native service discovery, PromQL | $0 (open source) |
| **Dashboards** | Grafana | Unified view across metrics, logs, traces | $0 (open source) |
| **Logging** | Grafana Loki + Promtail | Lightweight, label-indexed, pairs with Grafana | $0 (open source) |
| **Tracing** | Grafana Tempo + OpenTelemetry | Vendor-neutral instrumentation, trace-to-log correlation | $0 (open source) |
| **Alerting/On-Call** | PagerDuty or Opsgenie | Escalation policies, schedules, incident management | ~$500/month (team plan) |
| **Uptime/Synthetic** | Grafana synthetic monitoring or Uptime Kuma | External endpoint checks | $0 (self-hosted) |

### Budget Breakdown

| Item | Monthly Cost |
|------|-------------|
| Additional K8s nodes for monitoring workloads (2-3 nodes, ~8 vCPU / 32 GB each) | ~$600-900 |
| Persistent storage for metrics (Thanos + S3/GCS, ~500 GB) | ~$200-300 |
| Persistent storage for logs (Loki + object storage, ~1 TB) | ~$300-400 |
| Persistent storage for traces (Tempo + object storage, ~200 GB) | ~$50-100 |
| PagerDuty/Opsgenie team plan | ~$400-500 |
| Synthetic monitoring / external checks | ~$0-100 |
| **Buffer/contingency** | ~$500-700 |
| **Total** | **~$2,000-$3,000** |

> **Why not Datadog/New Relic?** At 12 microservices, Datadog would run $5K-15K/month (host-based + log ingestion + APM pricing). The open-source stack gives equivalent capability within your budget.

---

## 2. Architecture Overview

```
                    +-------------------+
                    |   Grafana (UI)    |
                    | Dashboards/Alerts |
                    +--------+----------+
                             |
              +--------------+--------------+
              |              |              |
     +--------v---+  +------v-----+  +-----v------+
     | Prometheus  |  | Loki       |  | Tempo      |
     | (Metrics)   |  | (Logs)     |  | (Traces)   |
     +--------+----+  +------+-----+  +-----+------+
              |              |              |
     +--------v---+  +------v-----+  +-----v------+
     | Thanos     |  | Promtail   |  | OTel       |
     | (Long-term |  | (Log       |  | Collector  |
     |  storage)  |  |  shipper)  |  | (Ingest)   |
     +--------+---+  +------+-----+  +-----+------+
              |              |              |
     +--------v--------------v--------------v------+
     |         Object Storage (S3 / GCS)           |
     +---------------------------------------------+

     +---------------------------------------------+
     |         12 Microservices (K8s Pods)          |
     |  - /metrics endpoint (Prometheus scrape)     |
     |  - stdout/stderr logs (Promtail DaemonSet)   |
     |  - OTel SDK instrumentation (traces)         |
     +---------------------------------------------+
```

---

## 3. Implementation Plan (4-Week Rollout)

### Week 1: Foundation -- Metrics and Dashboards

**Day 1-2: Deploy Prometheus via kube-prometheus-stack Helm chart**

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=7d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=100Gi \
  --set grafana.persistence.enabled=true \
  --set grafana.persistence.size=10Gi \
  --values custom-values.yaml
```

This single chart deploys:
- Prometheus (metrics collection)
- Grafana (dashboards)
- Alertmanager (alert routing)
- Node Exporter (host metrics)
- kube-state-metrics (K8s object metrics)

**Day 3-4: Instrument all 12 microservices**

Each service must expose a `/metrics` endpoint. Add a Prometheus client library:

- **Go**: `prometheus/client_golang`
- **Python**: `prometheus_client`
- **Node.js**: `prom-client`
- **Java/Kotlin**: Micrometer with Prometheus registry

Minimum metrics every service must expose:

```
# Request rate
http_requests_total{method, path, status_code}

# Request duration
http_request_duration_seconds{method, path}  (histogram)

# Errors
http_errors_total{method, path, error_type}

# In-flight requests
http_requests_in_flight

# Business-specific (examples)
orders_processed_total
queue_depth
cache_hit_ratio
```

Add Prometheus annotations to service deployments:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
spec:
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
```

**Day 5: Build core Grafana dashboards**

Create these dashboards (import community dashboards where possible, customize for your services):

1. **Cluster Overview** (Node health, pod status, resource utilization) -- import dashboard ID `315` or `1860`
2. **Service Overview** (RED metrics per service -- Rate, Errors, Duration) -- custom build
3. **Per-Service Deep Dive** (templated dashboard with service selector variable)
4. **Resource Usage** (CPU/memory requests vs. actual per namespace/service)

### Week 2: Logging

**Day 1-2: Deploy Loki + Promtail**

```bash
helm install loki grafana/loki \
  --namespace monitoring \
  --set loki.storage.type=s3 \
  --set loki.storage.s3.bucketnames=your-loki-bucket \
  --set loki.storage.s3.region=us-east-1 \
  --set loki.limits_config.retention_period=30d \
  --set loki.schemaConfig.configs[0].store=tsdb

helm install promtail grafana/promtail \
  --namespace monitoring \
  --set config.clients[0].url=http://loki-gateway.monitoring.svc:3100/loki/api/v1/push
```

**Day 3: Enforce structured logging across all services**

All 12 microservices must log in JSON format to stdout:

```json
{
  "timestamp": "2026-04-09T14:30:00.000Z",
  "level": "error",
  "service": "order-service",
  "trace_id": "abc123def456",
  "span_id": "789ghi",
  "message": "Failed to process payment",
  "error": "connection timeout",
  "user_id": "u-12345",
  "order_id": "ord-67890",
  "duration_ms": 3021
}
```

Key requirements:
- Always include `trace_id` for log-to-trace correlation
- Always include `level` for filtering
- Never log PII (mask credit cards, emails, etc.)
- Use consistent field names across all services

**Day 4-5: Configure Grafana log exploration and correlation**

- Add Loki as a Grafana data source
- Configure derived fields to link `trace_id` in logs to Tempo traces
- Build a "Service Logs" dashboard with log volume visualization and quick filters

### Week 3: Distributed Tracing

**Day 1-2: Deploy OpenTelemetry Collector + Tempo**

```bash
helm install tempo grafana/tempo-distributed \
  --namespace monitoring \
  --set storage.trace.backend=s3 \
  --set storage.trace.s3.bucket=your-tempo-bucket

helm install otel-collector open-telemetry/opentelemetry-collector \
  --namespace monitoring \
  --values otel-collector-values.yaml
```

OpenTelemetry Collector configuration (key parts):

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
    timeout: 5s
    send_batch_size: 1024
  tail_sampling:
    policies:
      - name: errors
        type: status_code
        status_code: { status_codes: [ERROR] }
      - name: slow-requests
        type: latency
        latency: { threshold_ms: 2000 }
      - name: probabilistic
        type: probabilistic
        probabilistic: { sampling_percentage: 10 }

exporters:
  otlp/tempo:
    endpoint: tempo-distributor.monitoring.svc:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch, tail_sampling]
      exporters: [otlp/tempo]
```

**Day 3-4: Instrument services with OpenTelemetry SDKs**

Add the OTel SDK to each microservice. Auto-instrumentation covers HTTP/gRPC/DB calls; add manual spans for business logic.

Example (Python):

```python
from opentelemetry import trace
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

provider = TracerProvider()
provider.add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter(endpoint="otel-collector.monitoring.svc:4317"))
)
trace.set_tracer_provider(provider)

# Auto-instrument frameworks
FastAPIInstrumentor.instrument()
RequestsInstrumentor.instrument()
SQLAlchemyInstrumentor().instrument()
```

**Day 5: Build trace dashboards and verify correlation**

- Add Tempo as a Grafana data source
- Configure Grafana to correlate: metrics -> traces (via exemplars), logs -> traces (via trace_id), traces -> logs
- Build a "Service Map" dashboard showing inter-service dependencies

### Week 4: Alerting, SLOs, and Operational Readiness

**Day 1-2: Define SLOs for each service**

Define SLIs and SLOs for every microservice:

| Service Tier | Availability SLO | Latency SLO (p99) | Error Budget (30d) |
|-------------|------------------|-------------------|-------------------|
| Tier 1 (user-facing, critical) | 99.9% | < 500ms | 43.2 minutes |
| Tier 2 (user-facing, non-critical) | 99.5% | < 1s | 3.6 hours |
| Tier 3 (internal/async) | 99.0% | < 5s | 7.2 hours |

Classify each of your 12 services into a tier. Example:
- Tier 1: API Gateway, Auth Service, Payment Service
- Tier 2: User Profile, Notification Service, Search
- Tier 3: Report Generator, Data Sync, Email Worker, etc.

**Day 2-3: Configure alerting rules**

Use multi-window burn-rate alerting (Google SRE approach) instead of static thresholds:

```yaml
# Example: SLO burn rate alert for a Tier 1 service (99.9% availability)
groups:
  - name: slo-burn-rate-alerts
    rules:
      # Fast burn (2% of 30-day budget in 1 hour) -- PAGE
      - alert: HighErrorBurnRate_Critical
        expr: |
          (
            sum(rate(http_requests_total{service="api-gateway", status_code=~"5.."}[1h]))
            /
            sum(rate(http_requests_total{service="api-gateway"}[1h]))
          ) > (14.4 * 0.001)
          and
          (
            sum(rate(http_requests_total{service="api-gateway", status_code=~"5.."}[5m]))
            /
            sum(rate(http_requests_total{service="api-gateway"}[5m]))
          ) > (14.4 * 0.001)
        for: 2m
        labels:
          severity: critical
          tier: "1"
        annotations:
          summary: "API Gateway burning error budget 14.4x faster than allowed"
          runbook_url: "https://wiki.internal/runbooks/api-gateway-errors"

      # Slow burn (10% of 30-day budget in 6 hours) -- TICKET
      - alert: HighErrorBurnRate_Warning
        expr: |
          (
            sum(rate(http_requests_total{service="api-gateway", status_code=~"5.."}[6h]))
            /
            sum(rate(http_requests_total{service="api-gateway"}[6h]))
          ) > (6 * 0.001)
          and
          (
            sum(rate(http_requests_total{service="api-gateway", status_code=~"5.."}[30m]))
            /
            sum(rate(http_requests_total{service="api-gateway"}[30m]))
          ) > (6 * 0.001)
        for: 5m
        labels:
          severity: warning
          tier: "1"
        annotations:
          summary: "API Gateway burning error budget 6x faster than allowed"
```

Additional alert categories to configure:

```yaml
# Infrastructure alerts
- KubePodCrashLooping (restart count > 3 in 10 min)
- KubePodNotReady (pod not ready > 5 min)
- KubeDeploymentReplicasMismatch (desired != available > 10 min)
- NodeDiskPressure
- NodeMemoryPressure
- PersistentVolumeFillingUp (< 15% free)

# Application alerts (per service)
- HighLatencyP99 (p99 > SLO threshold for tier)
- HighErrorRate (5xx rate > SLO budget burn rate)
- RequestQueueBacklog (in-flight > threshold)
- DatabaseConnectionPoolExhausted

# Monitoring stack self-monitoring
- PrometheusTargetDown
- PrometheusTSDBCompactionFailing
- LokiIngestionRateHigh
- AlertmanagerFailedNotifications
```

**Day 3: Set up PagerDuty/Opsgenie integration**

```yaml
# Alertmanager routing configuration
route:
  receiver: default-slack
  group_by: ['alertname', 'service', 'namespace']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - match:
        severity: critical
      receiver: pagerduty-critical
      continue: true
    - match:
        severity: warning
      receiver: slack-warnings

receivers:
  - name: pagerduty-critical
    pagerduty_configs:
      - service_key: '<your-pagerduty-integration-key>'
        severity: critical

  - name: slack-warnings
    slack_configs:
      - api_url: '<your-slack-webhook-url>'
        channel: '#alerts-warnings'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'

  - name: default-slack
    slack_configs:
      - api_url: '<your-slack-webhook-url>'
        channel: '#alerts-info'
```

**Day 4-5: Operational readiness**

1. **Runbooks**: Write a runbook for every critical and warning alert. Minimum content:
   - What the alert means
   - Impact assessment
   - Diagnostic steps (with exact commands)
   - Remediation steps
   - Escalation path

2. **On-call rotation**: Set up a rotation across the team with:
   - Primary and secondary on-call
   - Escalation after 5 minutes if unacknowledged
   - Weekly rotation to prevent burnout
   - Handoff documentation at rotation change

3. **Chaos/Load testing**: Before launch, validate monitoring works:
   - Kill a pod and verify alerting fires
   - Inject latency and verify SLO burn-rate alerts trigger
   - Send a burst of traffic and confirm dashboards show accurate data
   - Verify log-to-trace correlation works end to end

4. **Self-monitoring**: Ensure the monitoring stack monitors itself:
   - Prometheus scraping success rate
   - Loki ingestion lag
   - Tempo ingestion errors
   - Grafana health checks
   - Storage utilization trends

---

## 4. Key Metrics to Track per Microservice

### The Four Golden Signals (Google SRE)

| Signal | Metric | PromQL Example |
|--------|--------|---------------|
| **Latency** | Request duration by percentile | `histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{service="X"}[5m])) by (le))` |
| **Traffic** | Request rate | `sum(rate(http_requests_total{service="X"}[5m]))` |
| **Errors** | Error rate | `sum(rate(http_requests_total{service="X", status_code=~"5.."}[5m])) / sum(rate(http_requests_total{service="X"}[5m]))` |
| **Saturation** | Resource utilization | `container_memory_working_set_bytes / container_spec_memory_limit_bytes` |

### Additional Kubernetes-Specific Metrics

- Pod restart count (`kube_pod_container_status_restarts_total`)
- Pod scheduling latency
- HPA current vs. desired replicas
- PVC usage percentage
- Network I/O per pod

### Business Metrics (instrument per service)

- Orders processed per minute
- User sign-ups per hour
- Payment success/failure rate
- Queue depth and processing lag
- Cache hit ratio
- Database query latency (p50, p95, p99)

---

## 5. Dashboard Inventory

Build these dashboards in order of priority:

| # | Dashboard | Purpose | Audience |
|---|-----------|---------|----------|
| 1 | **Cluster Health** | Node status, pod health, resource utilization | SRE/Platform |
| 2 | **Service Overview (RED)** | Rate/Error/Duration for all 12 services on one screen | Everyone |
| 3 | **Per-Service Detail** | Deep dive with templated service variable | Dev teams |
| 4 | **SLO Burn Rate** | Error budget remaining per service per tier | SRE/Management |
| 5 | **Inter-Service Dependencies** | Service map with latency and error rates on edges | Architecture |
| 6 | **Log Explorer** | Structured log search with trace correlation | Dev teams |
| 7 | **Infrastructure** | PVCs, ingress, DNS, cert expiry | Platform |
| 8 | **Cost/Resource Efficiency** | Requests vs. actual usage, right-sizing candidates | FinOps |

---

## 6. Scaling Considerations

### Prometheus Scaling Strategy

With 12 microservices, a single Prometheus instance is likely sufficient initially. Plan for scaling when:

- **Cardinality exceeds 2M active series**: Shard Prometheus by namespace or team
- **Retention beyond 7 days needed**: Use Thanos with object storage for long-term metrics (already included in the architecture)
- **Query performance degrades**: Add Thanos Query Frontend with caching

### Loki Scaling

- Start with **SimpleScalable** deployment mode (3 read, 3 write, 1 backend)
- Set retention to 30 days (sufficient for debugging; archive to S3 for compliance)
- If log volume exceeds 100 GB/day, switch to microservices deployment mode

### Tempo Scaling

- Use **tail sampling** in the OTel Collector to control storage costs
- Sample 100% of errors and slow requests, 10% of normal traffic
- Adjust sampling rate based on actual trace volume and storage costs

---

## 7. Day-2 Operations Checklist

After launch, establish these ongoing practices:

- [ ] **Weekly**: Review SLO compliance and error budgets for all services
- [ ] **Weekly**: Check monitoring stack health (Prometheus memory, Loki ingestion, storage growth)
- [ ] **Monthly**: Review and tune alert thresholds based on false positive/negative rates
- [ ] **Monthly**: Archive old dashboards, update runbooks
- [ ] **Quarterly**: Review budget vs. actual spend; right-size monitoring infrastructure
- [ ] **Per-release**: Verify new services/endpoints have metrics, logs, and traces instrumented before shipping to production

---

## 8. Common Pitfalls to Avoid

1. **High-cardinality labels**: Never use user IDs, request IDs, or unbounded values as Prometheus labels. Use logs or traces for high-cardinality data.

2. **Alert fatigue**: Start with few, high-signal alerts (SLO burn rates). Add more only when you have a clear runbook. Target < 5 pages per on-call shift.

3. **Missing service dependencies**: If service A calls service B, instrument the client-side latency in A, not just the server-side in B. This captures network and queuing time.

4. **No baseline**: Run the monitoring stack for at least 1-2 weeks before setting alert thresholds. Use historical data, not guesses.

5. **Ignoring the monitoring stack itself**: Prometheus running out of disk or Loki falling behind on ingestion will blind you at the worst possible time. Self-monitoring is not optional.

6. **Log volume explosion**: Set ingestion rate limits in Loki. A single chatty microservice can blow your storage budget. Enforce log levels (no DEBUG in production by default).

---

## 9. Quick-Start Checklist

```
Phase 1 (Week 1):
  [x] Deploy kube-prometheus-stack (Prometheus + Grafana + Alertmanager)
  [x] Add /metrics endpoints to all 12 services
  [x] Build Cluster Health and Service RED dashboards
  [x] Configure basic alerts (pod crash loops, high error rates)

Phase 2 (Week 2):
  [x] Deploy Loki + Promtail
  [x] Enforce structured JSON logging across all services
  [x] Set up log-to-trace correlation in Grafana
  [x] Configure log retention and ingestion limits

Phase 3 (Week 3):
  [x] Deploy OTel Collector + Tempo
  [x] Add OTel SDK instrumentation to all services
  [x] Configure tail sampling
  [x] Build service dependency map

Phase 4 (Week 4):
  [x] Define SLOs per service tier
  [x] Implement burn-rate alerting
  [x] Set up PagerDuty/Opsgenie with routing
  [x] Write runbooks for all critical alerts
  [x] Chaos test the alerting pipeline
  [x] Self-monitoring verification
```

---

## 10. Helm Values Reference

For quick deployment, here is a consolidated `custom-values.yaml` for the kube-prometheus-stack:

```yaml
# custom-values.yaml for kube-prometheus-stack
prometheus:
  prometheusSpec:
    retention: 7d
    retentionSize: "80GB"
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3   # adjust for your cloud provider
          resources:
            requests:
              storage: 100Gi
    resources:
      requests:
        cpu: "1"
        memory: 4Gi
      limits:
        cpu: "2"
        memory: 8Gi
    # Scrape all pods with prometheus.io annotations
    additionalScrapeConfigs:
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port, __meta_kubernetes_pod_ip]
            action: replace
            target_label: __address__
            regex: (.+);(.+)
            replacement: $2:$1

grafana:
  persistence:
    enabled: true
    size: 10Gi
  adminPassword: "change-me-use-a-secret"
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
        - name: Loki
          type: loki
          url: http://loki-gateway.monitoring.svc:3100
          jsonData:
            derivedFields:
              - datasourceUid: tempo
                matcherRegex: '"trace_id":"(\w+)"'
                name: TraceID
                url: '$${__value.raw}'
        - name: Tempo
          type: tempo
          uid: tempo
          url: http://tempo-query-frontend.monitoring.svc:3100
          jsonData:
            tracesToLogs:
              datasourceUid: loki
              tags: ['service']
              mappedTags: [{ key: 'service.name', value: 'service' }]
              mapTagNamesEnabled: true
              filterByTraceID: true

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          resources:
            requests:
              storage: 10Gi

kubeStateMetrics:
  enabled: true

nodeExporter:
  enabled: true
```

---

This strategy gives you full-stack observability (metrics, logs, traces) with SLO-based alerting, all within your $3K/month budget. The open-source stack avoids vendor lock-in and scales well beyond 12 services. Focus on getting the Four Golden Signals instrumented in Week 1 -- that alone will give you significant visibility on launch day.
