# Challenge 2: Fix the Broken Deployment — Three Layered Issues

## 🎯 Challenge Goal
In namespace `t2`, the deployment `task-2` wants **3 healthy replicas** but all pods are failing.  
**Goal:** Get all 3 replicas running and ready.

---

## 🔴 Symptoms Observed

```bash
kubectl get deploy -n t2
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
task-2   0/3     2            0           24m
```

- Only 2 pods were being scheduled (not 3)
- Scheduled pods were stuck in `Pending` or `ImagePullBackOff`
- Zero replicas were READY

---

## 🔍 Investigation

**Tools used:**
```bash
kubectl describe pod/<pod-name> -n t2    # Pod-level events and errors
kubectl get node --show-labels           # Node label inspection
kubectl get quota -n t2                  # Namespace resource quota check
docker pull nginx:1.19-alpne             # Image tag verification
```

### Event stream from `kubectl describe pod`:
```
Warning  FailedScheduling   0/4 nodes are available: 1 node didn't match Pod's
                            node affinity/selector, 1 had untolerated taint,
                            2 were unschedulable.

Warning  Failed             Error: ImagePullBackOff
Warning  Failed             Failed to pull image "nginx:1.19-alpne":
                            docker.io/library/nginx:1.19-alpne: not found

Warning  FailedCreate       pods "task-2-..." is forbidden: exceeded quota:
                            tight-quota, requested: pods=1, requests.memory=64Mi,
                            used: pods=2, requests.memory=128Mi,
                            limited: pods=2, requests.memory=150Mi
```

---

## 🧠 Root Cause — 3 Independent Issues

### Issue 1 — Incorrect Node Selector
The deployment spec contained:
```yaml
nodeSelector:
  disk: ssd
```

No worker node had this label. Verified with:
```bash
kubectl get node --show-labels
# Result: none of the workers had disk=ssd label
```
The scheduler could not place pods → they stayed `Pending`.

---

### Issue 2 — Typo in Image Tag
The deployment used image `nginx:1.19-alpne` — a typo (`alpne` instead of `alpine`).

```bash
docker pull nginx:1.19-alpne
# Error: docker.io/library/nginx:1.19-alpne: not found

docker pull nginx:1.19-alpine
# 1.19-alpine: Pulling from library/nginx ✓
```

---

### Issue 3 — ResourceQuota Too Restrictive
The namespace `t2` had a quota (`tight-quota`) allowing only **2 pods** and **150Mi** memory.  
Three replicas at 64Mi each = 192Mi — exceeds the quota.

```bash
kubectl get quota -n t2
NAME          AGE   REQUEST                                   LIMIT
tight-quota   26m   pods: 2/2, requests.memory: 128Mi/150Mi
```

---

## 🔧 Fix Applied

### Fix 1 — Add the missing node label
```bash
kubectl label node sanjay-challenge-worker disk=ssd
```

### Fix 2 — Correct the image tag in the deployment
```bash
kubectl edit deployment task-2 -n t2
# Change: nginx:1.19-alpne
# To:     nginx:1.19-alpine
```

### Fix 3 — Update the ResourceQuota to allow 3 pods
```bash
kubectl edit resourcequota tight-quota -n t2
# Update: pods from 2 → 3
# Update: requests.memory from 150Mi → 200Mi (or higher)
```

---

## ✅ Verification

All 3 replicas running after applying all three fixes:

```bash
kubectl get pod -n t2
NAME                      READY   STATUS    RESTARTS   AGE
task-2-5b9fd4c864-2rbtg   1/1     Running   0          2s
task-2-5b9fd4c864-q2s6m   1/1     Running   0          3s
task-2-5b9fd4c864-rdw99   1/1     Running   0          2s
```

All 3 replicas READY. ✓

> **Key lesson:** Each issue masked the next one. Fix scheduling → pods try to pull image → image fails → fix tag → third pod blocked by quota → fix quota → all 3 run. Always investigate beyond the first error.
