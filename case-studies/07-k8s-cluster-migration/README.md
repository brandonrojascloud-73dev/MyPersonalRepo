# Kubernetes Cluster Migration: Zero Downtime at Scale

## Context

Our production Kubernetes cluster was running on an outdated version with known CVEs. The cloud provider announced end-of-life for the underlying infrastructure. We needed to migrate 200+ services to a new cluster without downtime.

## The Problem

### Constraints

- **Zero downtime**: SLA required 99.99% availability
- **200+ services**: StatefulSets, Deployments, Jobs, CronJobs
- **Stateful workloads**: PostgreSQL, Redis, Kafka with persistent volumes
- **External dependencies**: DNS, load balancers, secrets, certificates
- **Rollback requirement**: Must be able to revert within 5 minutes

### Why Not In-Place Upgrade?

- Cloud provider didn't support in-place control plane upgrade
- Node pool replacement would still require workload migration
- Risk of cascading failures during upgrade too high
- New cluster offered opportunity to improve architecture

## Architecture Decision

### Option 1: Big Bang Migration
Migrate everything at once during maintenance window.

**Pros**: Simple, fast
**Cons**: Violates zero-downtime requirement, high risk

### Option 2: Parallel Cluster with DNS Cutover
Run both clusters simultaneously, migrate services incrementally, switch DNS at the end.

**Pros**: Zero downtime, gradual rollout, easy rollback
**Cons**: Complex, requires dual-cluster operation, higher cost during migration

### Option 3: Blue-Green with Traffic Shifting
Similar to Option 2 but with service mesh controlling traffic split.

**Pros**: Fine-grained traffic control, canary deployments
**Cons**: Requires Istio/Linkerd, additional complexity

**Decision**: Option 2 (Parallel Cluster with DNS Cutover)

Rationale: Simpler than service mesh approach, meets zero-downtime requirement, proven pattern.

## Implementation

### Phase 1: New Cluster Setup (Week 1-2)

```bash
# New cluster with updated Kubernetes version
eksctl create cluster \
  --name production-v2 \
  --version 1.28 \
  --nodegroup-name standard-workers \
  --node-type m5.xlarge \
  --nodes-min 5 --nodes-max 50 \
  --managed
```

Key differences from old cluster:
- Kubernetes 1.28 (vs 1.24)
- New VPC with improved network architecture
- IRSA (IAM Roles for Service Accounts) instead of kube2iam
- Updated Calico network policies
- New EBS CSI driver (vs in-tree provider)

### Phase 2: State Migration (Week 2-3)

#### Secrets & ConfigMaps

```bash
# Export from old cluster
kubectl get secrets -n production -o json | \
  jq '.items[] | select(.type != "kubernetes.io/service-account-token")' > secrets.json

# Import to new cluster
cat secrets.json | kubectl apply -f -
```

For sensitive secrets, used external secrets operator with AWS Secrets Manager as source of truth.

#### Persistent Volumes

Stateful workloads required special handling:

**PostgreSQL**: Used logical replication
```sql
-- Old cluster
CREATE PUBLICATION migration_pub FOR ALL TABLES;

-- New cluster
CREATE SUBSCRIPTION migration_sub
  CONNECTION 'host=old-pg port=5432 dbname=app'
  PUBLICATION migration_pub;
```

**Redis**: Used redis-dump for initial sync, then enabled replication
```bash
redis-dump -u old-redis:6379 > redis-backup.json
redis-load -u new-redis:6379 < redis-backup.json
```

**Kafka**: Used MirrorMaker 2 for cross-cluster replication
```properties
# mm2.properties
us-east-1->us-west-2.enabled = true
us-east-1->us-west-2.topics = .*
```

### Phase 3: Workload Migration (Week 3-5)

Migrated services in waves based on dependency graph:

**Wave 1**: Stateless services with no external traffic (batch jobs, internal APIs)
**Wave 2**: Services with internal traffic only (backend APIs)
**Wave 3**: Services with external traffic (frontend, public APIs)

For each service:

```yaml
# Deploy to new cluster
kubectl apply -f deployment.yaml --context=new-cluster

# Verify health
kubectl wait --for=condition=ready pod -l app=my-service \
  --context=new-cluster --timeout=300s

# Run smoke tests
./smoke-tests.sh --cluster=new-cluster --service=my-service

# Update DNS (weighted routing)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456 \
  --change-batch file://dns-weight-50.json
```

DNS strategy:
- Start: 100% old cluster, 0% new
- Day 1: 90% old, 10% new (canary)
- Day 2: 50% old, 50% new
- Day 3: 0% old, 100% new

### Phase 4: Validation & Cutover (Week 5-6)

#### Monitoring

Deployed identical monitoring stack to both clusters:
- Prometheus + Grafana
- Datadog agents
- Custom health check dashboards comparing old vs new

```promql
# Comparison query
(
  sum(rate(http_requests_total{cluster="old"}[5m])) by (service)
  /
  sum(rate(http_requests_total{cluster="new"}[5m])) by (service)
)
```

#### Rollback Plan

At each phase, rollback was:

1. **DNS rollback**: Change Route53 weights back to 100% old cluster (< 60 seconds TTL)
2. **Data rollback**: For databases, keep old cluster running for 7 days post-migration
3. **Full rollback**: If critical issue found, revert DNS and investigate

## Challenges Solved

### 1. Service Discovery

Services communicating via Kubernetes DNS (e.g., `backend.production.svc.cluster.local`) needed to work across clusters during migration.

**Solution**: CoreDNS forwarding
```yaml
# Old cluster CoreDNS config
production.svc.cluster.local:53 {
    forward . NEW_CLUSTER_DNS_IP
}
```

### 2. Certificate Management

cert-manager issued certificates per-cluster. Services needed valid certs in both clusters.

**Solution**: Used same ACME account, but had to handle rate limits. Pre-issued certificates for new cluster 48 hours before cutover.

### 3. CronJob Coordination

CronJobs running in both clusters would execute twice.

**Solution**: Added cluster label to all CronJobs, used leader election for critical jobs:
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-report
  annotations:
    migration/active-cluster: "old"  # Updated during cutover
```

### 4. Persistent Volume Claims

PVCs bound to specific availability zones. New cluster had different AZ topology.

**Solution**: Created new PVCs in new cluster, used volume snapshot restore for initial data copy.

## Results

| Metric | Target | Actual |
|--------|--------|--------|
| Downtime | 0 minutes | 0 minutes |
| Migration duration | 6 weeks | 5.5 weeks |
| Services migrated | 200+ | 213 |
| Rollbacks needed | < 5% | 2% (4 services) |
| Cost during migration | +30% infrastructure | +28% |

## Key Learnings

1. **Start with stateless services.** They're easier to migrate and build confidence.

2. **DNS is your friend.** Weighted routing enables gradual cutover and instant rollback.

3. **Data migration is the hard part.** Stateless workloads are trivial; databases and queues require careful planning.

4. **Monitoring must be identical.** Can't compare what you can't measure.

5. **Keep old cluster running.** Don't decommission until you're 100% confident. We kept the old cluster for 10 days post-migration.

6. **Automate smoke tests.** Manual verification doesn't scale to 200+ services.

## Why This Matters

This demonstrates:
- **Complex migration planning**: Breaking a large migration into manageable phases
- **Risk mitigation**: Gradual cutover with instant rollback capability
- **Stateful workload expertise**: Database replication, queue migration, persistent storage
- **Operational maturity**: Monitoring, validation, rollback procedures
- **Zero-downtime execution**: Meeting strict SLA requirements during major infrastructure changes

Migrating a production Kubernetes cluster with 200+ services and zero downtime is the kind of high-stakes infrastructure work that separates senior engineers from the rest. It requires deep technical knowledge, careful planning, and the ability to execute under pressure.