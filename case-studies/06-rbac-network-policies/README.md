# RBAC & Network Policies: Zero Trust in Kubernetes

## Context

Our Kubernetes platform hosted 40+ microservices across multiple teams. The default configuration allowed any pod to talk to any other pod, and any service account had broad permissions. After a security audit flagged this, we implemented zero-trust networking and RBAC.

## The Problem

### Network: Flat Namespace

```
Before:
┌─────────────────────────────────────┐
│           Kubernetes Cluster         │
│                                      │
│  ┌─────────┐    ┌─────────┐        │
│  │ frontend │───▶│ backend │        │
│  └─────────┘    └────┬────┘        │
│       │               │              │
│       ▼               ▼              │
│  ┌─────────┐    ┌─────────┐        │
│  │  admin   │───▶│  database│        │
│  └─────────┘    └─────────┘        │
│                                      │
│  Every pod can reach every pod      │
└─────────────────────────────────────┘
```

### RBAC: Over-privileged Service Accounts

```yaml
# Before: cluster-admin for everything
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: default-admin
subjects:
- kind: ServiceAccount
  name: default
  namespace: *
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

## Implementation

### Phase 1: Network Policies (Deny All, Allow Specific)

**Step 1: Default deny all ingress**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}         # All pods
  policyTypes:
  - Ingress
  # No ingress rules = deny all ingress
```

Applied to every namespace. All inter-pod communication broke immediately. Then we added back only what was needed.

**Step 2: Service-specific allow rules**

```yaml
# Backend allows traffic only from frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-from-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - port: 8080
      protocol: TCP
```

```yaml
# Database allows traffic only from backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-from-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: database
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - port: 5432
      protocol: TCP
```

**Step 3: Cross-namespace policies**

```yaml
# Monitoring namespace can scrape all namespaces
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: production
spec:
  podSelector: {}
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - port: 9090
      protocol: TCP
```

### Phase 2: RBAC (Least Privilege)

**Step 1: Remove cluster-admin bindings**

```bash
# Audit existing bindings
kubectl get clusterrolebindings -o json | \
  jq -r '.items[] | select(.roleRef.name == "cluster-admin") | 
      .metadata.name + " -> " + (.subjects[].name // "unknown")'

# Delete overly broad bindings
kubectl delete clusterrolebinding default-admin
kubectl delete clusterrolebinding ci-deploy-admin
```

**Step 2: Create scoped roles**

```yaml
# Developer role: read-only in production, full in dev
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: development
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["pods", "deployments", "services", "configmaps", "secrets", "jobs"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: production
rules:
- apiGroups: ["", "apps"]
  resources: ["pods", "deployments", "services"]
  verbs: ["get", "list", "watch"]  # Read-only in prod
```

**Step 3: Service account isolation**

```yaml
# Each workload gets its own service account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/backend-irsa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: backend
  namespace: production
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["backend-config"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend
  namespace: production
subjects:
- kind: ServiceAccount
  name: backend
roleRef:
  kind: Role
  name: backend
  apiGroup: rbac.authorization.k8s.io
```

### Phase 3: Validation & CI Integration

Added network policy validation to CI pipeline:

```bash
# ci/validate-network-policies.sh
# Ensure every namespace has a default-deny policy

NAMESPACES=$(kubectl get ns -o jsonpath='{.items[*].metadata.name}')
MISSING=()

for ns in $NAMESPACES; do
  if [[ "$ns" == "kube-system" || "$ns" == "kube-public" ]]; then
    continue
  fi
  
  deny=$(kubectl get networkpolicy -n "$ns" -o json | \
    jq '[.items[] | select(.spec.podSelector == {} and 
                           (.spec.ingress // [] | length) == 0)] | length')
  
  if [[ "$deny" -eq 0 ]]; then
    MISSING+=("$ns")
  fi
done

if [[ ${#MISSING[@]} -gt 0 ]]; then
  echo "ERROR: Missing default-deny in: ${MISSING[*]}"
  exit 1
fi
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Pod-to-pod connectivity | Any-to-any | Explicit allow only |
| Service accounts with cluster-admin | 12 | 0 |
| Namespaces with default-deny | 0 | 100% |
| RBAC roles (principle of least privilege) | 3 (broad) | 28 (scoped) |
| Security audit findings (network/RBAC) | 14 | 0 |

## Challenges

1. **Breaking changes**: Default-deny broke existing services. We used a "log then deny" approach — deployed policies with audit logging first, analyzed traffic for 2 weeks, then enabled enforcement.

2. **DNS resolution**: Pods need to reach kube-dns. Added explicit egress policy:
```yaml
egress:
- to:
  - namespaceSelector:
      matchLabels:
        name: kube-system
  ports:
  - port: 53
    protocol: UDP
```

3. **Health checks**: Kubelet health checks need pod access. Added ingress from node CIDR for health check ports.

4. **Developer friction**: Developers couldn't `kubectl exec` in production anymore. Added a break-glass Role with time-limited bindings via an admission webhook.

## Why This Matters

This demonstrates:
- **Zero-trust implementation**: Not just talking about it — actually implementing deny-by-default networking and least-privilege RBAC
- **Incremental migration**: Breaking a large change into phases that don't break production
- **CI/CD integration**: Automated validation prevents regression
- **Security pragmatism**: Balancing security with developer productivity (break-glass roles)

The key insight: zero trust isn't a product you buy. It's an architecture you implement, one policy at a time.