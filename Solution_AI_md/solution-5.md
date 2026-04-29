# Challenge 5: TLS Certificate Debugging — Swapped CRT/Key + Wrong CA Bundle

## 🎯 Challenge Goal
In namespace `t5`, make this command succeed **without** `--insecure` or `-k`:
```bash
kubectl exec tls-client -n t5 -- curl --cacert /etc/ssl/custom/ca.crt \
  https://secure-app.t5.svc.cluster.local
```
Expected response: `TLS Challenge Complete!`

---

## 🔴 Symptoms Observed

**Symptom 1 — curl fails immediately:**
```bash
kubectl exec tls-client -n t5 -- curl --cacert /etc/ssl/custom/ca.crt \
  https://secure-app.t5.svc.cluster.local

curl: (7) Failed to connect to secure-app.t5.svc.cluster.local port 443
         after 128 ms: Could not connect to server
```

**Symptom 2 — `secure-app` pod is in CrashLoopBackOff:**
```bash
kubectl get pod -n t5
NAME                         READY   STATUS             RESTARTS
secure-app-d5d56dc59-5vzmh   0/1     CrashLoopBackOff   17 (3m27s ago)
tls-client                   1/1     Running            0
```

---

## 🔍 Investigation

### Step 1 — Check pod logs (note: `kubectl logs`, not `kubectl log`)
```bash
kubectl logs secure-app-d5d56dc59-5vzmh -n t5
```

**nginx startup error:**
```
2026/04/28 11:36:45 [emerg] 1#1: cannot load certificate "/etc/nginx/ssl/tls.crt":
PEM_read_bio_X509_AUX() failed
(SSL: error:0909006C:PEM routines:get_name:no start line:Expecting: TRUSTED CERTIFICATE)
```

nginx expects `tls.crt` to contain a certificate — but OpenSSL reported "no start line", meaning the file content does not begin with `-----BEGIN CERTIFICATE-----`.

### Step 2 — Decode the TLS secret to inspect the cert field:
```bash
kubectl get secret tls-secret -n t5 -o jsonpath='{.data.tls\.crt}' | base64 -d
```

**Output:**
```
-----BEGIN PRIVATE KEY-----
...
```

The `tls.crt` field contains a **private key**, not a certificate. The `tls.crt` and `tls.key` fields were **swapped** in the Kubernetes secret.

### Step 3 — Verify CA bundle mismatch (after fixing the secret):
After recreating the secret correctly, curl still failed with a certificate verification error, indicating the CA bundle mounted in `tls-client` did not match the CA that signed the server certificate.

---

## 🧠 Root Cause

| # | Issue | Detail |
|---|-------|--------|
| 1 | **Swapped cert and key in TLS secret** | `tls.crt` contained the private key; `tls.key` contained the certificate — nginx crashed on startup |
| 2 | **Wrong CA bundle in ConfigMap** | The CA cert mounted at `/etc/ssl/custom/ca.crt` inside `tls-client` was not the CA that signed `tls.crt` — curl rejected the server cert as untrusted |

---

## 🔧 Fix Applied

### Fix 1 — Recreate the TLS secret with correct field mapping

Extract the actual certificate and key from the existing secret (reversed):
```bash
# Get the actual cert (currently stored in tls.key)
kubectl get secret tls-secret -n t5 -o jsonpath='{.data.tls\.key}' | base64 -d > tls.crt

# Get the actual key (currently stored in tls.crt)
kubectl get secret tls-secret -n t5 -o jsonpath='{.data.tls\.crt}' | base64 -d > tls.key

# Delete and recreate the secret correctly
kubectl delete secret tls-secret -n t5
kubectl create secret tls tls-secret -n t5 --cert=tls.crt --key=tls.key
```

### Fix 2 — Update the CA bundle ConfigMap with the correct CA certificate

```bash
kubectl edit configmap ca-bundle -n t5
# Replace the ca.crt data with the correct CA certificate
# that was used to sign tls.crt
```

---

## ✅ Verification

```bash
kubectl exec tls-client -n t5 -- curl --cacert /etc/ssl/custom/ca.crt \
  https://secure-app.t5.svc.cluster.local
```

**Response:**
```
TLS Challenge Complete!
```

Full TLS handshake succeeded without `--insecure`. ✓

> **Key lesson:** When nginx crashes with `PEM_read_bio_X509_AUX() failed — Expecting: TRUSTED CERTIFICATE`, always decode the secret's `tls.crt` field first with `base64 -d`. If it shows `-----BEGIN PRIVATE KEY-----`, the cert and key are swapped. This is a common sabotage vector — and easy to miss because the secret looks "populated" from the outside.
