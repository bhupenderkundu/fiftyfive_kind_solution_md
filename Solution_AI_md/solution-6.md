# Challenge 6: Performance Triage Under Load — HPA Blocked by Missing Metrics Server

## 🎯 Challenge Goal
In namespace `t6`, the `api-server` deployment is experiencing intermittent failures under continuous load from a `load-generator` pod.

**Rules:**
- ❌ Do NOT stop or delete the `load-generator` pod
- ❌ Do NOT reduce the load generator's request rate

**Goal:** Make `api-server` handle all requests successfully with response times under 2 seconds. An HPA is configured but not working.

---

## 🔴 Symptoms Observed

- `api-server` pod was running (no crashes or restarts)
- Requests were failing / slow under load
- HPA was configured but replica count was not changing

```bash
kubectl get hpa -n t6
NAME      REFERENCE               TARGETS         MINPODS   MAXPODS   REPLICAS
api-hpa   Deployment/api-server   <unknown>/50%   1         5         1
```

`TARGETS` showing `<unknown>` — HPA cannot read CPU metrics.

---

## 🔍 Investigation

**Tool used:** `kubectl describe hpa api-hpa -n t6`

```
Conditions:
  Type           Status  Reason
  AbleToScale    True    SucceededGetScale
  ScalingActive  False   FailedGetResourceMetric

Events:
  Warning  FailedGetResourceMetric  (x381 over 102m)
  failed to get cpu utilization: unable to get metrics for resource cpu:
  unable to fetch metrics from resource metrics API:
  the server could not find the requested resource (get pods.metrics.k8s.io)
```

The HPA controller is attempting to call the `metrics.k8s.io` API — but that API does not exist in the cluster.

**Confirmed by:**
```bash
kubectl get pods -n kube-system | grep metrics-server
# (no output — metrics-server is not installed)

kubectl api-resources | grep metrics
# (no output — metrics.k8s.io API group not registered)
```

---

## 🧠 Root Cause

**`metrics-server` was not installed in the kind cluster.**

The Kubernetes HPA controller relies on the `metrics.k8s.io` API (provided by `metrics-server`) to retrieve pod CPU and memory utilization. Without `metrics-server`:

- The `metrics.k8s.io` API group does not exist
- HPA cannot compute a replica count
- `ScalingActive` condition stays `False`
- The deployment stays at minimum replicas regardless of load

Kind clusters do **not** ship with `metrics-server` pre-installed — it must be added manually.

---

## 🔧 Fix Applied

### Step 1 — Install metrics-server with kind-compatible flags

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Step 2 — Patch the deployment to add `--kubelet-insecure-tls`

Kind uses self-signed kubelet certificates. metrics-server needs this flag to work inside kind:

```bash
kubectl patch deployment metrics-server -n kube-system \
  --type='json' \
  -p='[{
    "op": "add",
    "path": "/spec/template/spec/containers/0/args/-",
    "value": "--kubelet-insecure-tls"
  }]'
```

### Step 3 — Wait for metrics-server to become ready

```bash
kubectl rollout status deployment metrics-server -n kube-system
# Waiting for deployment "metrics-server" rollout to finish...
# deployment "metrics-server" successfully rolled out
```

### Step 4 — Verify metrics API is now available

```bash
kubectl top nodes
kubectl top pods -n t6
```

---

## ✅ Verification

HPA now reads CPU metrics and scales correctly:

```bash
kubectl get hpa -n t6 -w
NAME      REFERENCE               TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
api-hpa   Deployment/api-server   cpu: 2%/50%   1         5         1          106m
```

`TARGETS` now shows actual CPU utilization (`2%/50%`) instead of `<unknown>`. HPA is active and will scale out when CPU crosses 50%. ✓

> **Key lesson:** HPA showing `<unknown>` for TARGETS almost always means `metrics-server` is missing or unhealthy. Always check `kubectl describe hpa` and look for `FailedGetResourceMetric` — then verify with `kubectl get pods -n kube-system | grep metrics-server` before assuming any application-level issue.
