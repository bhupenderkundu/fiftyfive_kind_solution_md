# Challenge 3: DNS Connectivity — CoreDNS ConfigMap Sabotage

## 🎯 Challenge Goal
From the `debug-client` pod in the `default` namespace, successfully run:
```bash
curl http://task-3.t3.svc.cluster.local
```
It must return the nginx welcome page.

---

## 🔴 Symptoms Observed

```bash
kubectl exec -it debug-client -- sh
~ # curl http://task-3.t3.svc.cluster.local
curl: (6) Could not resolve host: task-3.t3.svc.cluster.local
         (Domain name not found)
```

The service itself was healthy — correct ClusterIP, valid endpoint:

```bash
kubectl describe svc task-3 -n t3
Name:              task-3
Namespace:         t3
Type:              ClusterIP
IP:                10.108.14.27
Port:              80/TCP
TargetPort:        80/TCP
Endpoints:         10.244.1.5:80
```

Pod was running:
```bash
kubectl get pod -n t3
NAME                      READY   STATUS    RESTARTS   AGE
task-3-xxx                1/1     Running   0          33m
```

---

## 🔍 Investigation

**Tool used:** `kubectl exec` into `debug-client` + inspect CoreDNS ConfigMap

Since the service and endpoints were valid, the issue was clearly in DNS resolution.

**Step 1:** Verified CoreDNS pods were running:
```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
# Status: Running ✓
```

**Step 2:** Inspected the CoreDNS ConfigMap:
```bash
kubectl get configmap coredns -n kube-system -o yaml
```

Found a **malicious rewrite rule** in the Corefile:
```
ready
rewrite name task-3.t3.svc.cluster.local task-3.t3.svc.cluster.invalid
kubernetes cluster.local in-addr.arpa ip6.arpa {
    pods insecure
    fallthrough in-addr.arpa ip6.arpa
    ttl 30
}
```

---

## 🧠 Root Cause

A `rewrite` directive in the CoreDNS ConfigMap was intercepting DNS queries for `task-3.t3.svc.cluster.local` and redirecting them to `task-3.t3.svc.cluster.invalid` — a non-existent domain.

This caused every DNS lookup for that service to fail with NXDOMAIN, even though the service and pod were perfectly healthy. The sabotage was at the DNS layer, not the network layer.

---

## 🔧 Fix Applied

Edited the CoreDNS ConfigMap to remove the malicious rewrite line:

```bash
kubectl edit configmap coredns -n kube-system
```

**Removed this line:**
```
rewrite name task-3.t3.svc.cluster.local task-3.t3.svc.cluster.invalid
```

CoreDNS picks up ConfigMap changes automatically within a few seconds (no pod restart required).

---

## ✅ Verification

```bash
kubectl exec -it debug-client -- sh
~ # curl http://task-3.t3.svc.cluster.local
```

Response:
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>
</body>
</html>
```

DNS resolved correctly. Nginx welcome page returned. ✓

> **Key lesson:** When `curl: (6) Could not resolve host` appears for a service that clearly exists, always check the CoreDNS ConfigMap before assuming a pod or network issue. A single rewrite rule can silently black-hole traffic for a specific service.
