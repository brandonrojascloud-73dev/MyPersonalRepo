# Observability & SLO Implementation: From Guesswork to Measurable Reliability

## Context

Our platform had dashboards. We had alerts. We had logs. But we couldn't answer the most basic question: "Is the system reliable enough?"

We had no SLOs, no error budgets, and no objective measure of user experience. When leadership asked "are we improving?", the answer was always "I think so."

## The Problem

### What We Had
- 200+ Grafana dashboards (nobody looked at most of them)
- 500+ Prometheus alerts (most were noise)
- Logs in CloudWatch (expensive, rarely queried)
- No agreement on what "reliable" meant

### What We Needed
- **Measurable reliability targets** tied to user experience
- **Error budgets** that drive engineering decisions
- **Actionable alerts** that indicate real problems
- **Observability** that answers questions, not just shows graphs

## Implementation

### Phase 1: Define SLIs (Service Level Indicators)

For each user-facing service, we identified what "working" means from the user's perspective:

```yaml
# SLI definitions
service: api-gateway
sli_indicators:
  
  availability:
    description: "Percentage of non-5xx responses"
    good: response_code < 500
    total: all_requests
    target: 99.9%
    
  latency:
    description: "Response time for successful requests"
    good: latency_ms < 500
    total: successful_requests  
    target: p99 < 500ms
    
  freshness:
    description: "Data is less than N seconds old"
    good: data_age_seconds < 30
    target: 99% of queries
```

Implementation in Prometheus:

```promql
# Availability SLI
(
  sum(rate(http_requests_total{job="api-gateway", code!~"5.."}[5m]))
  /
  sum(rate(http_requests_total{job="api-gateway"}[5m]))
) * 100

# Latency SLI (p99)
histogram_quantile(0.99, 
  sum(rate(http_request_duration_seconds_bucket{job="api-gateway"}[5m])) by (le)
) * 1000
```

### Phase 2: Define SLOs (Service Level Objectives)

SLOs are targets for SLIs over a time window:

```yaml
# SLO definitions
slos:
  api-gateway-availability:
    sli: api-gateway/availability
    target: 99.9%
    window: 30d
    
  api-gateway-latency:
    sli: api-gateway/latency
    target: p99 < 500ms
    window: 30d
    
  backend-throughput:
    sli: backend/requests-per-second
    target: "> 1000 rps"
    window: 1h
```

### Phase 3: Error Budgets

Error budget = 100% - SLO target. It quantifies how much unreliability is acceptable:

```
SLO: 99.9% availability over 30 days
Error budget: 0.1% = 43.2 minutes of downtime allowed per month
```

Error budget policy:

```yaml
error_budget_policy:
  api-gateway:
    slo: 99.9%
    window: 30d
    budget_minutes: 43.2
    
    thresholds:
      - name: "healthy"
        budget_remaining: "> 50%"
        action: "normal development, feature work OK"
        
      - name: "caution"
        budget_remaining: "25-50%"
        action: "reduce change velocity, focus on reliability"
        
      - name: "critical"
        budget_remaining: "< 25%"
        action: "freeze non-critical changes, reliability-only PRs"
        
      - name: "exhausted"
        budget_remaining: "< 0%"
        action: "stop all feature work, incident response mode"
```

### Phase 4: Alerting on SLO Burn Rate

Instead of alerting on symptoms (CPU, memory), we alert on **SLO burn rate** — how fast we're consuming the error budget:

```promql
# Burn rate alert: consuming error budget 14x faster than sustainable
(
  1 - (
    sum(rate(http_requests_total{job="api-gateway", code!~"5.."}[5m]))
    /
    sum(rate(http_requests_total{job="api-gateway"}[5m]))
  )
)
/
(1 - 0.999)  # Error budget = 0.1%
> 14.4  # 14.4x burn rate = exhausts 30-day budget in ~2 days
```

Alert rule:
```yaml
groups:
  - name: slo-alerts
    rules:
      - alert: HighErrorBudgetBurn
        expr: |
          (
            1 - (
              sum(rate(http_requests_total{code!~"5.."}[5m]))
              / sum(rate(http_requests_total[5m]))
            )
          ) / (1 - 0.999) > 14.4
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "API Gateway burning error budget at 14.4x rate"
          description: "At current rate, 30-day error budget will be exhausted in ~2 days"
```

### Phase 5: Observability Stack

#### Metrics (Prometheus + Grafana)

```yaml
# Recording rules for efficiency
groups:
  - name: slo-recording
    interval: 30s
    rules:
      - record: job:sli_availability:ratio_rate5m
        expr: |
          sum(rate(http_requests_total{code!~"5.."}[5m])) by (job)
          / sum(rate(http_requests_total[5m])) by (job)
          
      - record: job:sli_availability:error_budget_remaining
        expr: |
          (job:sli_availability:ratio_rate5m - 0.999) / (1 - 0.999)
```

Grafana dashboard panels:
1. **SLI current value** (gauge)
2. **Error budget remaining** (progress bar)
3. **Burn rate over time** (graph)
4. **SLO compliance history** (calendar heatmap)

#### Logs (Structured + Contextual)

Moved from unstructured logs to structured JSON with trace context:

```json
{
  "timestamp": "2024-01-15T10:23:45.123Z",
  "level": "error",
  "service": "api-gateway",
  "trace_id": "abc123def456",
  "span_id": "span789",
  "user_id": "user-456",
  "request": {
    "method": "POST",
    "path": "/api/v2/orders",
    "latency_ms": 1234
  },
  "error": {
    "code": "DATABASE_TIMEOUT",
    "message": "Query exceeded 5s timeout",
    "stack": "..."
  },
  "sli_impact": {
    "availability": false,
    "latency": true
  }
}
```

#### Traces (OpenTelemetry)

```python
# Instrumentation example
from opentelemetry import trace

tracer = trace.get_tracer("api-gateway")

@tracer.start_as_current_span("process_order")
def process_order(order_id):
    span = trace.get_current_span()
    span.set_attribute("order.id", order_id)
    
    with tracer.start_as_current_span("validate_inventory") as child:
        # ... inventory check
        child.set_attribute("inventory.available", True)
        
    with tracer.start_as_current_span("charge_payment") as child:
        # ... payment processing
        child.set_attribute("payment.amount", 99.99)
```

## Implementation Details

### Multi-Window Burn Rate Alerts

Single-window alerts produce false positives. We implemented multi-window alerts:

```yaml
# Short window (1h): detects rapid burn
# Long window (6h): confirms sustained problem

alerts:
  - name: CriticalBurnRate
    expr: |
      # 1h window: burning 14.4x rate
      burn_rate_1h > 14.4
      and
      # 6h window: burning 6x rate  
      burn_rate_6h > 6
    for: 2m
    severity: critical
    
  - name: WarningBurnRate
    expr: |
      # 1h window: burning 6x rate
      burn_rate_1h > 6
      and
      # 6h window: burning 3x rate
      burn_rate_6h > 3
    for: 5m
    severity: warning
```

### SLO Reporting

Automated weekly reports to engineering leads:

```
SLO Report - Week of Jan 15, 2024

Service: api-gateway
  Availability SLO: 99.9% | Actual: 99.95% | Budget remaining: 87%
  Latency SLO: p99 < 500ms | Actual: p99 423ms | Budget remaining: 92%
  
Service: backend
  Availability SLO: 99.5% | Actual: 99.3% | Budget remaining: 42% ⚠️
  Latency SLO: p95 < 200ms | Actual: p95 187ms | Budget remaining: 78%

Actions required:
  - backend team: reduce change velocity, focus on reliability improvements
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Can answer "is system reliable?" | No | Yes, with data |
| Meaningful alerts | ~10% | ~85% |
| Time to detect reliability degradation | Hours | Minutes |
| Engineering decisions based on data | Rarely | Always |
| Error budget exhaustion incidents | 4/quarter | 1/quarter |

## Lessons Learned

1. **Start with user-facing services.** Internal services matter, but users experience the edge first.

2. **SLOs are conversations, not numbers.** The value is in agreeing what "reliable enough" means, not the specific target.

3. **Burn rate alerts beat threshold alerts.** Alerting on "error rate > 1%" produces noise. Alerting on "consuming budget 14x faster" produces action.

4. **Error budgets change behavior.** When teams know they have 43 minutes of downtime budget per month, they think differently about risky deployments.

5. **Observability is a product.** Treat it like one: users (engineers), features (dashboards, alerts), iteration (feedback loops).

## Why This Matters

This demonstrates:
- **Measurable reliability**: Moving from "I think" to "the data shows"
- **Error budget-driven decisions**: Engineering prioritization based on reliability data
- **Modern alerting**: Burn rate alerts over threshold alerts
- **Full observability stack**: Metrics + logs + traces with correlation
- **Cultural change**: Reliability as a shared, measurable goal

The key insight: you can't improve what you can't measure. SLOs give you the measurement. Error budgets give you the decision framework. Observability gives you the ability to debug when things go wrong.