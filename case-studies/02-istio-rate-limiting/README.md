# Istio Rate Limiting: Configured but Never Working

## The Problem

Rate limiting was configured in our Istio service mesh since 2021. Multiple PRs attempted fixes. The rate limit service was running. The EnvoyFilter was applied. But **no requests were ever rate limited**.

We discovered this during a security audit. The oauth endpoints had no protection against brute force attacks.

## Initial Investigation

The architecture looked correct:
1. EnvoyFilter `ratelimit` installed the HTTP filter
2. EnvoyFilter `ratelimit-actions` defined rate limit rules
3. EnvoyFilter `ratelimit-route-config` enabled rate limiting per route
4. Rate limit service running in `ratelimit` namespace

But when we checked Envoy stats:
```bash
kubectl exec -n istio-system $POD -- curl -s localhost:15000/stats | grep ratelimit
```

Zero activity. No `ok`, no `over_limit`, no `error` counters. The filter was installed but never executed.

## False Leads

We chased several red herrings:

1. **Port confusion**: The rate limit service listens on 8081 (gRPC), but we saw 444k requests to port 8080. Those were health checks, not rate limiting.

2. **Path matching**: We thought the oauth path `/public/v2/oauth/token` wasn't matching. Added `/public/v3/oauth/token`. Still nothing.

3. **Virtual host configuration**: We removed an empty `name: ""` from vhost match. Actions started applying to vhosts, but filter still didn't work.

Each "fix" gave us false hope. The real problem was hiding in plain sight.

## Root Cause

The rate limit HTTP filter was **installed but never enabled**.

Without explicit `filter_enabled` configuration, Envoy falls back to a runtime key: `ratelimit.http_filter_enabled`. Istio doesn't configure Envoy Runtime by default. Without runtime config AND without explicit `filter_enabled`, the filter defaults to **DISABLED**.

The filter was in the chain. Actions were configured. The service was running. But Envoy never checked any requests.

## The Fix

Add these fields to the `ratelimit` EnvoyFilter:

```yaml
filter_enabled:
  default_value:
    numerator: 100
    denominator: HUNDRED
filter_enforced:
  default_value:
    numerator: 100
    denominator: HUNDRED
```

- `filter_enabled`: Percentage of requests checked by the rate limit filter (100% = all requests)
- `filter_enforced`: Percentage of enabled requests that enforce the decision (100% = full enforcement)

These fields enable gradual rollout (10%, 50%, 100%) and shadow mode testing (enable but don't enforce).

## Validation

After deploying the fix:

1. **Config dump verification**:
```bash
kubectl exec -n istio-system $POD -- curl -s localhost:15000/config_dump | \
  jq '.configs[] | select(.\"@type\" == \"type.googleapis.com/envoy.admin.v3.ListenersConfigDump\") | 
      .dynamic_listeners[].active_state.listener | 
      select(.name | contains(\"0.0.0.0_8080\")) | 
      .filter_chains[0].filters[0].typed_config.http_filters[] | 
      select(.name == \"envoy.filters.http.ratelimit\") | 
      .typed_config.value | 
      {has_filter_enabled: has(\"filter_enabled\")}'
```
Result: `true`

2. **Traffic test**:
```bash
for i in {1..15}; do
  curl -s -o /dev/null -w "Request $i: HTTP %{http_code}\n" \
    -X POST https://api-devt.example.com/public/v2/oauth/token \
    -H "Content-Type: application/json" \
    -d '{"grant_type":"client_credentials"}'
  sleep 1
done
```

Result: First 10 requests returned HTTP 400 (invalid credentials). Requests 11-15 returned **HTTP 429** (rate limited).

3. **Envoy stats**:
```
cluster.outbound|8081||ratelimit.ratelimit.svc.cluster.local.ratelimit.ok: 10
cluster.outbound|8081||ratelimit.ratelimit.svc.cluster.local.ratelimit.over_limit: 5
cluster.outbound|8081||ratelimit.ratelimit.svc.cluster.local.ratelimit.error: 0
```

## Lessons Learned

1. **Defaults are dangerous.** Envoy's default behavior (disable filter without explicit config) is a safety mechanism, but it's invisible until you know to look for it.

2. **Stats don't lie.** Zero counters meant the filter never executed. We should have trusted that signal earlier.

3. **Runtime config is a trap.** Relying on Envoy Runtime for filter enablement adds complexity. Explicit config is clearer.

4. **Production incidents teach.** A separate incident where rate limiting was accidentally deployed to production taught us to always verify `kubectl context` before applying changes.

## Why This Matters

Rate limiting is a security control. Having it configured but not enforced is worse than not having it at all—it creates false confidence. This case shows the importance of **validating that security controls actually work**, not just that they're configured.

The fix took 6 months of intermittent investigation, 4 failed PR attempts, and one security audit to prioritize. That's how infrastructure debt accumulates.