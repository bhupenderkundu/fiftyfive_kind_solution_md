# 🎙️ VIDEO DIALOGUE SCRIPT
## DevOps Hiring Challenge — All 6 Solutions
**Style:** Natural, confident, conversational — like explaining to a senior engineer peer  
**Estimated Duration:** 20–25 minutes

---

## ▶️ INTRO
*[Show your terminal or a title card]*

"Hi, I'm Bhupender. In this video I'm walking through all six challenges from the DevOps hiring assignment.

For each challenge, I'll cover the same four points — what I saw, how I investigated, what the root cause was, and how I fixed and proved it.

Let's get into it."

---

---

## 🔧 CHALLENGE 1 — Terraform Cluster Setup
*[Open terminal at the terraform directory]*

---

### 🗣️ Opening

"Challenge 1 is setting up the kind cluster using Terraform. The first thing I always do with a new Terraform config is run `terraform plan` — not apply, just plan — to surface any issues before touching real infrastructure."

---

### 🗣️ Symptoms

"When I ran `terraform plan`, I got two errors back to back.

The first one said — *Invalid value for path parameter: no file exists at `./../kubernetes/cluster-config.yaml`*.

The second one said — *Reference to undeclared input variable: `kube_config_path`*.

Two separate problems. Let me show you what caused each one."

---

### 🗣️ Root Cause — Error 1

*[Show the file listing: `ls ../kubernetes/`]*

"The Terraform code was calling `file()` with the path `cluster-config.yaml`. But when I listed the actual kubernetes directory, the file on disk was `kind-config.yaml`. Different name entirely.

The `file()` function in Terraform is strict — it only works with files that exist on disk at plan time. If the name doesn't match exactly, it fails. This is a classic copy-paste error — someone wrote the config name from memory instead of checking the actual filename."

---

### 🗣️ Root Cause — Error 2

*[Show main.tf line 42]*

"The second error was on line 42 of `main.tf`. The command was using `var.kube_config_path`. But when I checked `variables.tf`, the variable was declared as `kind_config_path` — with `kind`, not `kube`.

Terraform doesn't try to guess what you meant. If the variable name in the reference doesn't match the declaration exactly, it treats it as undeclared and throws an error."

---

### 🗣️ Fix

"Both fixes were simple one-liners in `main.tf`:
- Changed `cluster-config.yaml` to `kind-config.yaml`
- Changed `var.kube_config_path` to `var.kind_config_path`

That's it."

---

### 🗣️ Verification

*[Run terraform plan → clean output. Then terraform apply]*

"Terraform plan ran clean. Then apply completed in about a minute and a half.

The output showed the cluster came up with one control plane and three workers — `sanjay-challenge-worker`, `worker2`, and `worker3`. All four challenge namespaces were created. We're ready for the next challenges."

---

---

## 🔧 CHALLENGE 2 — Fix the Broken Deployment
*[kubectl context active, namespace t2]*

---

### 🗣️ Opening

"Challenge 2 is about getting three healthy replicas of a deployment called `task-2` in namespace `t2`. The hint said there are multiple issues — and there were. Three of them, each one hiding the next."

---

### 🗣️ Symptoms

*[Run `kubectl get deploy -n t2`]*

"Running `kubectl get deploy -n t2` showed zero out of three ready. Only two pods were even being scheduled, not three. So something was already wrong before we got to the image or the application.

I ran `kubectl describe pod` on one of the stuck pods to get the event stream."

---

### 🗣️ Root Cause — Issue 1: Node Selector

*[Show `kubectl get node --show-labels`]*

"The first event was a scheduling failure: *zero out of four nodes available — one didn't match the pod's node affinity selector*.

I looked at the deployment spec and found a `nodeSelector` requiring `disk: ssd`. Then I ran `kubectl get node --show-labels` and checked all the workers — none of them had that label. The scheduler had nowhere to put the pods, so they just sat there in Pending.

Fix: I added the label to the worker node — `kubectl label node sanjay-challenge-worker disk=ssd`. Once the label was there, pods could be scheduled."

---

### 🗣️ Root Cause — Issue 2: Image Tag Typo

*[Show the ImagePullBackOff event, then `docker pull nginx:1.19-alpne`]*

"Once pods started scheduling, the next error showed up — `ImagePullBackOff`. The image in the deployment was `nginx:1.19-alpne`. Look carefully — a-l-p-n-e. That's a typo. The correct tag is `nginx:1.19-alpine`.

I confirmed it by running `docker pull nginx:1.19-alpne` — not found. Then `docker pull nginx:1.19-alpine` — pulled fine.

Fix: Edited the deployment to use the correct tag `nginx:1.19-alpine`."

---

### 🗣️ Root Cause — Issue 3: ResourceQuota

*[Show the FailedCreate event and `kubectl get quota -n t2`]*

"Now with two issues fixed, I expected three pods. But the third one still wouldn't create. The replicaset controller event said — *pods is forbidden: exceeded quota tight-quota, requested pods=1, used pods=2, limited pods=2*.

The namespace had a ResourceQuota called `tight-quota` — and it only allowed two pods total. Three replicas at 64Mi each also exceeded the memory limit.

Fix: I edited the ResourceQuota to allow three pods and increase the memory ceiling."

---

### 🗣️ Verification

*[Run `kubectl get pod -n t2`]*

"After all three fixes, all three replicas came up running. The key takeaway here is that each issue was masking the next. You can't see the image problem until scheduling works, and you can't see the quota problem until the image pulls. You have to keep investigating each error until there are no more errors."

---

---

## 🔧 CHALLENGE 3 — DNS Connectivity
*[kubectl exec into debug-client]*

---

### 🗣️ Opening

"Challenge 3 was a connectivity problem. From a pod called `debug-client` in the default namespace, I needed to curl `task-3` in namespace `t3`. The hint said there are multiple blocking layers."

---

### 🗣️ Symptoms

*[Show the curl failure]*

"When I exec'd into `debug-client` and ran the curl, I got — *curl error 6: Could not resolve host: task-3.t3.svc.cluster.local*.

Error 6 from curl means DNS resolution failed entirely. The host was never even looked up — it just came back as unknown."

---

### 🗣️ Investigation

*[Show `kubectl describe svc task-3 -n t3`]*

"My first check was the service itself. Described it — ClusterIP assigned, endpoint pointing to the pod IP on port 80. The service was fine. The pod was running. Network policy wasn't blocking anything obvious.

So the problem was in DNS. I checked whether CoreDNS pods were healthy — they were running normally.

Then I looked at the CoreDNS ConfigMap."

---

### 🗣️ Root Cause

*[Show the Corefile rewrite line]*

"Found it. In the Corefile there was this line:

```
rewrite name task-3.t3.svc.cluster.local task-3.t3.svc.cluster.invalid
```

This is a CoreDNS rewrite directive. It was intercepting every DNS query for that specific service name and redirecting it to `.cluster.invalid` — which doesn't exist anywhere. Every lookup returned NXDOMAIN. The service was healthy, the network was fine — but DNS was being deliberately poisoned at the config level."

---

### 🗣️ Fix and Verification

*[Run kubectl edit configmap coredns -n kube-system, then re-run curl]*

"Fix was simple — open the ConfigMap, delete that one rewrite line, save. CoreDNS picks up ConfigMap changes automatically, no pod restart needed.

Ran the curl again from `debug-client` — got the full nginx welcome page back. DNS resolved, connection made, response received."

---

---

## 🔧 CHALLENGE 4 — Node Recovery
*[kubectl get node showing worker2 NotReady]*

---

### 🗣️ Opening

"Challenge 4 is node recovery. `sanjay-challenge-worker2` was stuck in `NotReady`. The hint told us the node runs as a Docker container and to use `docker exec` to get inside."

---

### 🗣️ Investigation

*[kubectl describe node, then docker exec]*

"I ran `kubectl describe node` on worker2 and the condition showed `KubeletNotReady — kubelet stopped posting status`. That tells me the kubelet process itself is down, not just a workload issue.

Since this is a kind cluster, each node is actually a Docker container. I exec'd directly into it:

```bash
docker exec -it sanjay-challenge-worker2 bash
```

Checked the kubelet service — it was in a failed state. Checked the journal logs:

```bash
journalctl -u kubelet -n 50
```

The error was — *unable to read client-cert `/var/lib/kubelet/pki/kubelet-client-current.pem`: no such file or directory*.

The kubelet couldn't load its TLS client certificate, so it couldn't authenticate to the API server, so it crashed."

---

### 🗣️ Root Cause

*[Show `ls -la /var/lib/kubelet/pki/`]*

"I listed the PKI directory. The actual cert file was there — `kubelet-client-2026-04-28-10-31-11.pem`. But the symlink that the kubelet always reads from — `kubelet-client-current.pem` — had been renamed to `kubelet-client-current.pem.bak`.

Someone had just added `.bak` to the symlink name, which broke the kubelet's fixed reference path. The cert existed but the pointer to it was gone."

---

### 🗣️ Fix and Verification

*[Show the ln -s command, then systemctl restart kubelet]*

"Fix: recreate the symlink pointing to the actual cert file.

```bash
ln -s /var/lib/kubelet/pki/kubelet-client-2026-04-28-10-31-11.pem \
      /var/lib/kubelet/pki/kubelet-client-current.pem
```

Then restart kubelet. Exited the container, ran `kubectl get node` — worker2 came back `Ready` within about 30 seconds."

---

---

## 🔧 CHALLENGE 5 — TLS Certificate Debugging
*[namespace t5]*

---

### 🗣️ Opening

"Challenge 5 was full TLS debugging. The task was to make a curl command work against an HTTPS nginx endpoint using a custom CA certificate — without the `-k` insecure flag. That means the full TLS chain has to be valid."

---

### 🗣️ Symptoms

*[Show the curl failure and pod status]*

"Running the curl gave — *Failed to connect to secure-app port 443: Could not connect to server*.

Checked the pods — `secure-app` was in `CrashLoopBackOff` with 17 restarts. So the server wasn't even up. That's the first problem to solve."

---

### 🗣️ Root Cause — Issue 1: CrashLoopBackOff

*[kubectl logs the pod]*

"I ran `kubectl logs` on the crashing pod. Note — it's `kubectl logs`, not `kubectl log`. Got this error from nginx:

*cannot load certificate `/etc/nginx/ssl/tls.crt`: PEM_read_bio_X509_AUX() failed — Expecting: TRUSTED CERTIFICATE*

nginx is saying the file it's reading as a certificate doesn't look like a certificate. I decoded the TLS secret to see what's actually in there:

```bash
kubectl get secret tls-secret -n t5 -o jsonpath='{.data.tls\.crt}' | base64 -d
```

Output started with `-----BEGIN PRIVATE KEY-----`.

The `tls.crt` field contained the private key. The `tls.key` field contained the certificate. They were swapped inside the Kubernetes secret.

Fix: deleted the secret and recreated it with `tls.crt` and `tls.key` in the correct positions."

---

### 🗣️ Root Cause — Issue 2: CA Bundle Mismatch

"After the secret fix, the pod came up. But curl still failed — this time with a certificate verification error. That means the TLS handshake was happening but the client didn't trust the server's cert.

The `tls-client` pod had a CA bundle mounted at `/etc/ssl/custom/ca.crt` via a ConfigMap. That CA bundle didn't match the CA that had actually signed the server certificate.

Fix: updated the ConfigMap with the correct CA certificate."

---

### 🗣️ Verification

*[Run the full curl command]*

"Ran the curl again with `--cacert`. Got back — `TLS Challenge Complete!`

Full handshake, no insecure flag, correct response. Both issues had to be fixed for the end-to-end TLS chain to work."

---

---

## 🔧 CHALLENGE 6 — Performance Triage Under Load
*[namespace t6, HPA not scaling]*

---

### 🗣️ Opening

"Challenge 6 was performance under load. The `api-server` deployment had an HPA configured to scale up to 5 replicas when CPU hit 50%. But it wasn't scaling, even with a load generator running continuously. The rules were clear — don't touch the load generator."

---

### 🗣️ Symptoms

*[kubectl get hpa -n t6]*

"The first thing I checked was the HPA status. The TARGETS column showed `<unknown>/50%` instead of an actual CPU percentage.

`<unknown>` means the HPA controller can't get any metrics data. It's not a scaling threshold issue — the HPA literally doesn't know what the CPU is."

---

### 🗣️ Investigation and Root Cause

*[kubectl describe hpa, showing FailedGetResourceMetric event]*

"I ran `kubectl describe hpa api-hpa -n t6`. The conditions block showed `ScalingActive: False`, reason `FailedGetResourceMetric`. The event message said:

*failed to get cpu utilization: unable to fetch metrics from resource metrics API: the server could not find the requested resource (get pods.metrics.k8s.io)*

The `metrics.k8s.io` API doesn't exist in this cluster. I confirmed it:

```bash
kubectl get pods -n kube-system | grep metrics-server
```

No output. `metrics-server` was never installed.

Kind clusters don't come with `metrics-server` out of the box. The HPA was configured correctly, but without the metrics API, it was completely blind — it couldn't compute a replica count, so it never scaled."

---

### 🗣️ Fix

*[Show the apply command and the patch command]*

"Fix was two steps.

First, install metrics-server from the official manifest:

```bash
kubectl apply -f https://github.com/.../metrics-server/.../components.yaml
```

Second — and this is kind-specific — patch the deployment to add `--kubelet-insecure-tls`. Kind uses self-signed kubelet certificates, so metrics-server needs that flag to scrape kubelet metrics without TLS verification errors.

```bash
kubectl patch deployment metrics-server -n kube-system \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```"

---

### 🗣️ Verification

*[kubectl get hpa -n t6 -w]*

"Once metrics-server came up, the HPA started reporting actual CPU — `cpu: 2%/50%`. It was now active and watching. Under load spikes, it would scale out automatically.

Challenge 6 — done."

---

---

## ▶️ OUTRO
*[Return to terminal or title card]*

---

### 🗣️ Summary

"So that's all six challenges wrapped up. Quick summary:

Challenge 1 was Terraform — a filename mismatch and a variable name mismatch. Fixed with two line changes in `main.tf`.

Challenge 2 was a broken deployment with three stacked issues — wrong node selector, typo in the image tag, and a ResourceQuota that was too tight. All three had to be fixed in sequence.

Challenge 3 was DNS — a malicious rewrite rule in the CoreDNS ConfigMap was silently black-holing queries for one specific service.

Challenge 4 was node recovery — the kubelet PKI symlink had been renamed to `.bak`, so kubelet couldn't load its certificate and crashed on every start.

Challenge 5 was TLS — the cert and key were swapped inside the Kubernetes secret, crashing nginx, and the CA bundle in the client pod didn't match the server's CA.

Challenge 6 was HPA not scaling because `metrics-server` was never installed in the kind cluster — the metrics API didn't exist.

Thanks for watching."

---

*Script by Bhupender Kundu | DevOps Hiring Assignment*
