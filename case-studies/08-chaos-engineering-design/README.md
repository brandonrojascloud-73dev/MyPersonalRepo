# Chaos Engineering: Designing Failure Scenarios for Resilience Validation

## Context

Our platform had 99.95% uptime on paper. But we had never validated what would happen under real failure conditions. After a cascading outage took down 3 services for 45 minutes, leadership approved a chaos engineering program.

## The Problem

Most teams think chaos engineering is "randomly kill pods." In production, that's how you cause outages, not prevent them.

What we actually needed:
- **Controlled experiments** with measurable outcomes
- **Blast radius containment** so experiments don't cascade
- **Reproducible scenarios** that teams can run in staging before production
- **Clear success/failure criteria** tied to SLOs

## Methodology: The Experiment Framework

### Experiment Design Template

Every chaos experiment follows this structure:

```yaml
experiment:
  name: "database-primary-failover"
  hypothesis: "Application continues serving read requests within 5s when DB primary fails"
  
  preconditions:
    - service: api-gateway
      health: healthy
    - service: database
      replicas: 3
      primary: db-primary-0
    
  steady_state:
    metric: http_success_rate
    target: "> 99.9%"
    window: 60s
    
  fault:
    type: pod-kill
    target: db-primary-0
    namespace: database
    
  expected_behavior:
    - metric: http_success_rate
      threshold: "> 95%"
      duration: 30s
    - metric: db_failover_time
      threshold: "< 10s"
    - metric: error_rate
      threshold: "< 5%"
      duration: 60s
      
  rollback:
    trigger: "http_success_rate < 90% for 30s"
    action: "restore db-primary-0"
```

### Categories of Failure Scenarios

We designed experiments across 6 failure domains:

**1. Infrastructure Failures**
```
- Node termination (graceful vs ungraceful)
- AZ failure (simulate entire AZ going down)
- Network partition (service A cannot reach service B)
- Disk I/O latency injection
- CPU/memory pressure
```

**2. Application Failures**
```
- Pod crash loop
- OOM kill
- Leader election failure
- Health check failure
- Slow response (latency injection)
```

**3. Dependency Failures**
```
- Database primary failover
- Cache eviction (Redis flush)
- Queue broker unavailability
- External API timeout
- DNS resolution failure
```

**4. Deployment Failures**
```
- Rolling update stuck (new pods never ready)
- Bad configmap rollout
- Certificate expiration
- Image pull failure
```

**5. Data Failures**
```
- Corrupted configuration
- Stale cache serving wrong data
- Message queue poison pill
- Database connection pool exhaustion
```

**6. Security Failures**
```
- Service account token expired
- IAM role permission revoked
- Network policy blocking legitimate traffic
- Certificate CN mismatch
```

## Implementation

### Tooling: Custom Chaos Controller

We built a lightweight chaos controller rather than adopting a heavy framework:

```python
# chaos_controller.py (simplified)
class ChaosExperiment:
    def __init__(self, spec: ExperimentSpec):
        self.spec = spec
        self.results = []
        
    def validate_preconditions(self) -> bool:
        """Ensure system is in expected state before fault injection"""
        for precond in self.spec.preconditions:
            if not self.check_service_health(precond.service):
                return False
        return True
    
    def establish_steady_state(self) -> Metrics:
        """Capture baseline metrics before fault"""
        return self.collect_metrics(
            metrics=self.spec.steady_state.metric,
            window=self.spec.steady_state.window
        )
    
    def inject_fault(self):
        """Apply the fault and start monitoring"""
        fault = self.spec.fault
        
        if fault.type == "pod-kill":
            self.k8s.delete_pod(fault.target, fault.namespace)
        elif fault.type == "network-partition":
            self.apply_network_policy(fault.target, deny=fault.deny)
        elif fault.type == "latency-inject":
            self.add_tc_rules(fault.target, delay_ms=fault.delay_ms)
            
    def monitor(self) -> ExperimentResult:
        """Monitor expected_behavior and detect pass/fail"""
        timeout = self.spec.expected_behavior[0].duration
        start = time.time()
        
        while time.time() - start < timeout:
            metrics = self.collect_metrics()
            
            # Check rollback trigger
            if self.should_rollback(metrics):
                self.rollback()
                return ExperimentResult(status="ROLLED_BACK", metrics=metrics)
            
            # Check expected behavior
            if not self.meets_threshold(metrics):
                return ExperimentResult(status="FAILED", metrics=metrics)
                
            time.sleep(5)
            
        return ExperimentResult(status="PASSED", metrics=metrics)
```

### Blast Radius Containment

Critical rule: **experiments must not cascade beyond the target**.

```yaml
# Containment strategy per environment
environments:
  staging:
    max_affected_services: 3
    require_manual_approval: false
    allow_production_impact: false
    
  production:
    max_affected_services: 1
    require_manual_approval: true
    allow_production_impact: false
    business_hours_only: true
    rollback_timeout: 60s
```

Implementation via Kubernetes admission webhook:
```go
// webhook validates experiment spec against containment rules
func validateExperiment(exp *ChaosExperiment) error {
    affected := calculateBlastRadius(exp)
    
    if len(affected) > exp.Environment.MaxAffectedServices {
        return fmt.Errorf("blast radius %d exceeds limit %d", 
            len(affected), exp.Environment.MaxAffectedServices)
    }
    
    if exp.Environment.Name == "production" && !exp.ManualApproval {
        return fmt.Errorf("production experiments require manual approval")
    }
    
    return nil
}
```

## Example Experiment: Queue Consumer Failure

### Scenario
Message queue has 10,000 unprocessed messages. Consumer pods start crashing. What happens?

### Expected Behaviors
1. Messages remain in queue (not lost)
2. Dead letter queue captures poison messages after 3 retries
3. Consumer auto-scales from 3 to 10 pods
4. Processing resumes within 30s of pod recovery
5. No duplicate processing (idempotency)

### Intentional Defects (for training)
We created 5 variants with different bugs:

| Variant | Defect | Expected Diagnosis |
|---------|--------|-------------------|
| A | Consumer doesn't NACK on failure | Messages stuck in-flight, not retried |
| B | No idempotency key | Duplicate processing after retry |
| C | Dead letter queue misconfigured | Poison messages block queue |
| D | Autoscaler metric wrong (CPU vs queue depth) | No scale-up under load |
| E | Consumer acks before processing | Message loss on crash |

Each variant is a fully reproducible environment that can be used for:
- Training engineers on debugging distributed systems
- Evaluating AI model ability to diagnose infrastructure issues
- Validating monitoring catches the specific failure mode

## Results

| Metric | Before | After |
|--------|--------|-------|
| Mean time to detect cascading failures | 23 min | 4 min |
| Runbooks with validated recovery procedures | 30% | 95% |
| Incidents caused by untested failure modes | 3-4/quarter | 0-1/quarter |
| Team confidence in failover mechanisms | Low | High |

## Why This Matters

This demonstrates:
- **Systematic failure thinking**: Not random chaos — structured experiments with hypotheses
- **Reproducible defect design**: Creating variants that test specific failure modes
- **Containment engineering**: Ensuring experiments don't cause real outages
- **Measurable outcomes**: Every experiment has pass/fail criteria tied to SLOs
- **Training value**: Defective variants serve as learning environments

The key insight: chaos engineering isn't about breaking things. It's about **proving your system handles breaks correctly** — and building confidence through evidence, not hope.