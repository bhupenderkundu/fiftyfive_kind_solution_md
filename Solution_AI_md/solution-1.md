# Challenge 1: Terraform Cluster Setup — File & Variable Name Mismatch

## 🎯 Challenge Goal
Provision a `kind` Kubernetes cluster using Terraform. The configuration had intentional bugs that prevented `terraform plan` from succeeding.

---

## 🔴 Symptoms Observed

Running `terraform plan` immediately produced **two errors**:

### Error 1 — Invalid file path
```
Error: Invalid function argument

  on main.tf line 36, in resource "null_resource" "kind_cluster":
  36:     kind_config = sha256(file("${path.module}/../kubernetes/cluster-config.yaml"))

Invalid value for "path" parameter: no file exists at
"./../kubernetes/cluster-config.yaml"; this function works only with files
that are distributed as part of the configuration source code.
```

### Error 2 — Undeclared input variable
```
Error: Reference to undeclared input variable

  on main.tf line 42, in resource "null_resource" "kind_cluster":
  42:       kind create cluster --name ${var.cluster_name} --config ${var.kube_config_path}

An input variable with the name "kube_config_path" has not been declared.
This variable can be declared with a variable "kube_config_path" {} block.
```

---

## 🔍 Investigation

**Tool used:** `terraform plan`

This is the standard first step to validate configuration and surface all errors before any infrastructure is created. It showed both issues simultaneously.

```bash
cd ~/devops-hiring-assignment/terraform
terraform plan
```

To confirm the actual filename on disk:
```bash
ls ../kubernetes/
# Output: kind-config.yaml   ← actual filename
```

---

## 🧠 Root Cause

| # | Problem | Detail |
|---|---------|--------|
| 1 | **Filename mismatch** | `main.tf` referenced `cluster-config.yaml` but the actual file on disk was `kind-config.yaml` |
| 2 | **Variable name mismatch** | `main.tf` used `var.kube_config_path` but the variable was declared as `kind_config_path` in `variables.tf` |

Both are naming inconsistencies — the code does not match the actual files and variable declarations.

---

## 🔧 Fix Applied

**Fix 1 — Correct the file path reference in `main.tf`:**
```hcl
# Before (broken)
kind_config = sha256(file("${path.module}/../kubernetes/cluster-config.yaml"))

# After (fixed)
kind_config = sha256(file("${path.module}/../kubernetes/kind-config.yaml"))
```

**Fix 2 — Correct the variable reference in `main.tf` line 42:**
```hcl
# Before (broken)
kind create cluster --name ${var.cluster_name} --config ${var.kube_config_path}

# After (fixed)
kind create cluster --name ${var.cluster_name} --config ${var.kind_config_path}
```

---

## ✅ Verification

`terraform plan` ran cleanly with zero errors. Then `terraform apply` completed successfully:

```
null_resource.sabotage: Creation complete after 1m22s [id=5670533291352604169]

Apply complete! Resources: 4 added, 0 changed, 1 destroyed.

Outputs:

challenge_namespaces = tolist([
  "t2",
  "t3",
  "t5",
  "t6",
])
cluster_name    = "sanjay-challenge"
cluster_nodes   = {
  "control_plane" = "sanjay-challenge-control-plane"
  "workers" = [
    "sanjay-challenge-worker",
    "sanjay-challenge-worker2",
    "sanjay-challenge-worker3",
  ]
}
```

Cluster provisioned with 1 control plane node and 3 worker nodes. ✓
