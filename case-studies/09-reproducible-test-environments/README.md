# Reproducible Infrastructure Test Environments: Golden Solutions & Defective Variants

## Context

Our platform team needed a way to:
1. Validate infrastructure changes before production
2. Train new engineers on real-world scenarios
3. Benchmark automation tooling against known-good configurations
4. Create reproducible environments for incident response practice

The solution: a library of containerized infrastructure environments with reference solutions and intentionally defective variants.

## The Problem

Infrastructure debugging is hard to practice because:
- Production environments are unique and constantly changing
- Setting up realistic scenarios takes hours/days
- There's no "answer key" to validate against
- Defects are discovered organically, not designed intentionally

We needed **reproducible environments** where:
- A scenario can be spun up in minutes
- The correct solution is documented and validated
- Variants with specific defects can be created deterministically
- Tests can verify if the environment was fixed correctly

## Architecture

### Environment Specification

Each test environment is defined as a self-contained package:

```
environments/
├── k8s-deployment-rollback/
│   ├── golden/                    # Reference solution (working state)
│   │   ├── manifests/
│   │   ├── validation.sh
│   │   └── solution.md
│   ├── variants/                  # Defective versions
│   │   ├── bad-image-tag/
│   │   ├── missing-configmap/
│   │   ├── resource-limits-oom/
│   │   └── readiness-probe-fail/
│   ├── scenario.yaml              # Environment specification
│   └── README.md
│
├── database-replication-lag/
│   ├── golden/
│   ├── variants/
│   │   ├── network-partition/
│   │   ├── slow-disk/
│   │   └── lock-contention/
│   ├── scenario.yaml
│   └── README.md
│
└── ...
```

### Scenario Specification

```yaml
# scenario.yaml
name: k8s-deployment-rollback
category: deployment
difficulty: intermediate
time_estimate: 15m

infrastructure:
  runtime: kind  # Kubernetes in Docker
  k8s_version: "1.28"
  nodes: 3
  
services:
  - name: api-gateway
    image: nginx:1.25
    replicas: 3
    resources:
      requests: { cpu: 100m, memory: 128Mi }
      limits: { cpu: 500m, memory: 256Mi }
    
  - name: backend
    image: backend:v2.1.0
    replicas: 2
    configmap: backend-config
    secrets: db-credentials
    
  - name: database
    image: postgres:15
    replicas: 1
    persistent_volume:
      size: 1Gi
      storage_class: standard

validation:
  - name: "all pods running"
    command: "kubectl get pods --field-selector=status.phase=Running | wc -l"
    expected: "6"
    
  - name: "api reachable"
    command: "curl -sf http://api-gateway/health"
    expected_exit: 0
    
  - name: "database connected"
    command: "kubectl exec backend-0 -- pg_isready -h database"
    expected_exit: 0

golden_solution:
  description: "All services deployed and healthy with correct configuration"
  setup_script: "golden/setup.sh"
  
defective_variants:
  - name: bad-image-tag
    description: "Backend references non-existent image tag"
    mutation: "golden/manifests/backend.yaml: change image to backend:v9.9.9"
    expected_symptoms: "ImagePullBackOff, deployment stuck"
    expected_fix: "Correct image tag or rollback to previous version"
    
  - name: missing-configmap
    description: "ConfigMap not created before deployment"
    mutation: "golden/manifests/configmap.yaml: delete"
    expected_symptoms: "Pod stuck in ContainerCreating, event: configmap not found"
    expected_fix: "Create configmap or fix reference"
```

## Implementation

### Environment Builder

```python
# environment_builder.py
class EnvironmentBuilder:
    def __init__(self, scenario_path: str):
        self.scenario = yaml.safe_load(open(f"{scenario_path}/scenario.yaml"))
        self.runtime = self._init_runtime()
        
    def _init_runtime(self):
        """Initialize containerized runtime (kind, docker-compose, etc.)"""
        if self.scenario["infrastructure"]["runtime"] == "kind":
            return KindRuntime(
                k8s_version=self.scenario["infrastructure"]["k8s_version"],
                nodes=self.scenario["infrastructure"]["nodes"]
            )
        # ... other runtimes
    
    def build_golden(self) -> Environment:
        """Build the reference solution (working state)"""
        env = self.runtime.create()
        
        # Apply golden manifests
        golden_path = f"{self.scenario_path}/golden/manifests"
        env.kubectl_apply(golden_path)
        
        # Wait for validation
        for check in self.scenario["validation"]:
            env.wait_for(check, timeout=120)
            
        return env
    
    def build_variant(self, variant_name: str) -> Environment:
        """Build a defective variant"""
        variant = next(
            v for v in self.scenario["defective_variants"] 
            if v["name"] == variant_name
        )
        
        env = self.build_golden()  # Start from golden
        
        # Apply mutation
        env.apply_mutation(variant["mutation"])
        
        return env
    
    def validate(self, env: Environment) -> ValidationResult:
        """Run validation checks against environment"""
        results = []
        for check in self.scenario["validation"]:
            output = env.execute(check["command"])
            passed = output.strip() == check["expected"]
            results.append(CheckResult(
                name=check["name"],
                passed=passed,
                expected=check["expected"],
                actual=output.strip()
            ))
        return ValidationResult(results=results)
```

### Mutation Engine

The mutation engine applies deterministic defects:

```python
# mutations.py
class MutationEngine:
    @staticmethod
    def apply(manifest_path: str, mutation: str):
        """Apply a mutation to Kubernetes manifests"""
        # Parse mutation instruction
        if "change image to" in mutation:
            new_image = mutation.split("change image to ")[1]
            return MutationEngine._change_image(manifest_path, new_image)
        elif "delete" in mutation:
            return MutationEngine._delete_file(manifest_path)
        elif "modify resource" in mutation:
            return MutationEngine._modify_resource(manifest_path, mutation)
        # ... more mutation types
    
    @staticmethod
    def _change_image(manifest_path: str, new_image: str):
        """Change container image in deployment manifest"""
        doc = yaml.safe_load(open(manifest_path))
        doc["spec"]["template"]["spec"]["containers"][0]["image"] = new_image
        yaml.dump(doc, open(manifest_path, "w"))
    
    @staticmethod  
    def _inject_network_latency(namespace: str, target: str, delay_ms: int):
        """Inject network latency using tc/netem"""
        pod = k8s.get_pod(namespace, target)
        k8s.exec(pod, f"tc qdisc add dev eth0 root netem delay {delay_ms}ms")
```

## Example: Database Replication Lag Scenario

### Golden Solution
```yaml
# Working state: primary-replica with < 1s lag
primary:
  image: postgres:15
  config:
    max_wal_senders: 5
    wal_level: replica
    
replica:
  image: postgres:15
  config:
    hot_standby: "on"
    primary_conninfo: "host=primary port=5432"

validation:
  - name: "replication lag < 1s"
    command: "SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp()))::int"
    expected: "< 1"
```

### Variant: Slow Disk I/O
```yaml
mutation:
  type: disk-latency
  target: primary
  delay_ms: 500
  
expected_symptoms:
  - "Replication lag increases to 10-30s"
  - "Read queries on replica return stale data"
  - "Application timeout on write-heavy operations"
  
expected_diagnosis:
  - "Check pg_stat_replication on primary"
  - "Identify high wal_write wait events"
  - "Correlate with disk I/O metrics"
  
expected_fix:
  - "Move to faster storage (io2 EBS)"
  - "Or increase wal_sender_timeout"
  - "Or add read replica to distribute read load"
```

## Validation Framework

### Deterministic Tests

Each environment has tests that verify the solution:

```bash
#!/bin/bash
# validation.sh for k8s-deployment-rollback scenario

set -e

# Test 1: All pods are running
POD_COUNT=$(kubectl get pods -l app=backend --field-selector=status.phase=Running -o json | jq '.items | length')
if [ "$POD_COUNT" -ne 2 ]; then
    echo "FAIL: Expected 2 running backend pods, got $POD_COUNT"
    exit 1
fi

# Test 2: Deployment is not in failed state
ROLLOUT_STATUS=$(kubectl rollout status deployment/backend --timeout=5s 2>&1)
if echo "$ROLLOUT_STATUS" | grep -q "failed"; then
    echo "FAIL: Deployment rollout failed"
    exit 1
fi

# Test 3: API gateway returns 200
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://api-gateway/health)
if [ "$HTTP_CODE" -ne 200 ]; then
    echo "FAIL: API gateway health check returned $HTTP_CODE"
    exit 1
fi

# Test 4: Database is accessible
kubectl exec -it backend-0 -- pg_isready -h database -U postgres
if [ $? -ne 0 ]; then
    echo "FAIL: Cannot connect to database"
    exit 1
fi

echo "PASS: All validation checks passed"
```

### Automated Scoring

```python
# scorer.py
class EnvironmentScorer:
    def score(self, env: Environment, solution_attempts: list) -> Score:
        """Score solution attempts against golden validation"""
        
        # Run validation checks
        validation = self.builder.validate(env)
        
        # Calculate score
        checks_passed = sum(1 for c in validation.results if c.passed)
        total_checks = len(validation.results)
        
        # Time penalty (optional)
        time_taken = solution_attempts[-1].timestamp - solution_attempts[0].timestamp
        time_penalty = max(0, (time_taken - self.scenario["time_estimate"]) / 60)
        
        return Score(
            accuracy=checks_passed / total_checks,
            time_penalty=time_penalty,
            final_score=max(0, (checks_passed / total_checks) - time_penalty * 0.1)
        )
```

## Usage Patterns

### 1. Pre-Production Validation
```bash
# Before applying changes to production, validate in test environment
env = EnvironmentBuilder("environments/k8s-deployment-rollback")
test_env = env.build_golden()
test_env.apply_change(production_change_manifest)
result = env.validate(test_env)
if not result.all_passed:
    print("Change would break validation - aborting")
    sys.exit(1)
```

### 2. Engineer Training
```bash
# New engineer practices debugging
env = EnvironmentBuilder("environments/k8s-deployment-rollback")
defective = env.build_variant("bad-image-tag")
# Engineer diagnoses and fixes the issue
# Automated validation confirms fix
```

### 3. AI Model Evaluation
```bash
# Feed defective environment to AI model
env = EnvironmentBuilder("environments/k8s-deployment-rollback")
defective = env.build_variant("missing-configmap")

# Model receives: pod events, logs, kubectl output
# Model proposes: create configmap with correct data
# Validation confirms: all checks pass → model solved it
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Time to set up training scenario | 2-4 hours | 3 minutes |
| Scenarios available for practice | 0 | 47 |
| New engineer onboarding time | 3 weeks | 10 days |
| Production incidents from untested changes | 8/quarter | 1/quarter |
| Validation coverage for infrastructure changes | 20% | 90% |

## Why This Matters

This demonstrates:
- **Reproducible environment design**: Containerized, versioned, deterministic
- **Golden reference solutions**: Known-good state for comparison
- **Intentional defect creation**: Mutations that produce specific, diagnosable failures
- **Automated validation**: Objective pass/fail criteria
- **Training at scale**: Environments available on-demand for practice

The key insight: you can't improve what you can't reproduce. By making infrastructure scenarios reproducible, you enable practice, validation, and continuous improvement — for both humans and AI systems.