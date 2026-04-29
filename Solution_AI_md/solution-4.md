# Challenge 4: Node Recovery — Broken kubelet PKI Symlink

## 🎯 Challenge Goal
Node `sanjay-challenge-worker2` has gone `NotReady`.  
**Goal:** Bring it back to `Ready` status so it can schedule and run pods.

---

## 🔴 Symptoms Observed

```bash
kubectl get node
NAME                             STATUS                        ROLES           AGE   VERSION
sanjay-challenge-control-plane   Ready                         control-plane   9m    v1.32.2
sanjay-challenge-worker          Ready,SchedulingDisabled      <none>          8m    v1.32.2
sanjay-challenge-worker2         NotReady,SchedulingDisabled   <none>          8m    v1.32.2
sanjay-challenge-worker3         Ready                         <none>          8m    v1.32.2
```

`sanjay-challenge-worker2` showing `NotReady`.

---

## 🔍 Investigation

**Tools used:**
```bash
kubectl describe node sanjay-challenge-worker2   # Node conditions & events
docker exec -it sanjay-challenge-worker2 bash    # Enter the kind node container
systemctl status kubelet                          # kubelet service status
journalctl -u kubelet -n 50                       # kubelet logs
ls -la /var/lib/kubelet/pki/                      # PKI directory inspection
```

### Step 1 — Node describe showed kubelet not posting status:
```
Conditions:
  Type    Status   Reason
  Ready   False    KubeletNotReady — kubelet stopped posting status
```

### Step 2 — Exec into the kind node (Docker container):
```bash
docker exec -it sanjay-challenge-worker2 bash
```

### Step 3 — kubelet service was crashed:
```bash
systemctl status kubelet
# Active: failed (Result: exit-code)
```

### Step 4 — kubelet journal logs revealed the root cause:
```
bootstrap.go:241] "Unhandled Error" err="unable to read existing bootstrap
client config from /etc/kubernetes/kubelet.conf: invalid configuration:
[unable to read client-cert /var/lib/kubelet/pki/kubelet-client-current.pem:
no such file or directory, unable to read client-key
/var/lib/kubelet/pki/kubelet-client-current.pem:
no such file or directory]"

run.go:72] "command failed" err="failed to run Kubelet: unable to load
bootstrap kubeconfig: stat /etc/kubernetes/bootstrap-kubelet.conf:
no such file or directory"
```

### Step 5 — PKI directory listing confirmed the sabotage:
```bash
ls -la /var/lib/kubelet/pki/
# Output:
-rw------- kubelet-client-2026-04-28-10-31-11.pem    ← actual cert file (exists)
lrwxrwxrwx kubelet-client-current.pem.bak            ← symlink renamed to .bak!
-rw-r--r-- kubelet.crt
-rw------- kubelet.key
```

---

## 🧠 Root Cause

The symlink `kubelet-client-current.pem` had been **renamed to `kubelet-client-current.pem.bak`**.

The kubelet always reads its client certificate from the fixed path `kubelet-client-current.pem`. The actual `.pem` cert file existed, but because the symlink was gone, the kubelet crashed on every start attempt — it could not authenticate to the API server.

---

## 🔧 Fix Applied

**Step 1 — Recreate the symlink inside the node container:**
```bash
ln -s /var/lib/kubelet/pki/kubelet-client-2026-04-28-10-31-11.pem \
      /var/lib/kubelet/pki/kubelet-client-current.pem
```

**Step 2 — Restart the kubelet:**
```bash
systemctl restart kubelet
systemctl status kubelet
# Active: active (running) ✓
```

**Step 3 — Exit the container:**
```bash
exit
```

---

## ✅ Verification

```bash
kubectl get node
NAME                             STATUS                     ROLES           AGE   VERSION
sanjay-challenge-control-plane   Ready                      control-plane   67m   v1.32.2
sanjay-challenge-worker          Ready,SchedulingDisabled   <none>          67m   v1.32.2
sanjay-challenge-worker2         Ready,SchedulingDisabled   <none>          67m   v1.32.2  ✓
sanjay-challenge-worker3         Ready                      <none>          67m   v1.32.2
```

`sanjay-challenge-worker2` returned to `Ready`. ✓

> **Key lesson:** In kind clusters, nodes are Docker containers. When a node goes `NotReady`, always `docker exec` into it and check `journalctl -u kubelet`. A missing or renamed symlink in the PKI directory is enough to completely crash the kubelet — even if all cert files are intact.
