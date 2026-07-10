# AWS EKS Upgrade Runbook (Template)

Use this document for every EKS upgrade. Fill in the tables and follow each step in order.

## Upgrade record — fill in before starting

| Field | Value |
|---|---|
| Cluster name | |
| Environment | |
| Region | |
| Current version | |
| Target version | |
| Upgrade date | |
| Owner | |
| Change ticket | |

---

## 1. Objective

Upgrade the EKS cluster from the current version to the next supported version.

| Rule | Detail |
|---|---|
| One version at a time | EKS does not support skipping minor versions |
| Example | 1.32 → 1.34 requires two upgrades: 1.32 → 1.33, then 1.33 → 1.34 |

---

## 2. Pre-Migration Checks

Complete all checks before making any changes.

| # | Check | What to do | Done |
|---|---|---|---|
| 1 | API compatibility | Run Pluto tool. Fix deprecated APIs before upgrade | [ ] |
| 2 | Applications | Confirm apps and Helm charts work on target version | [ ] |
| 3 | Pod Disruption Budgets | Ensure PDBs do not block node draining | [ ] |
| 4 | StatefulSets | List StatefulSets. Check storage and PV requirements | [ ] |
| 5 | Webhooks | Verify Validating/Mutating webhooks are healthy | [ ] |
| 6 | Probes | Confirm readiness and liveness probes are set correctly | [ ] |
| 7 | Cluster health | All nodes Ready. All kube-system pods healthy | [ ] |
| 8 | Add-ons | Check EKS and third-party add-ons support target version | [ ] |
| 9 | Backup | Back up manifests, Helm values, and config files | [ ] |

---

## 3. Where to Update Version in Terraform

| Item | File / path |
|---|---|
| Cluster version | `tf-script/configs/<environment>.yml` → `eks.eks_cluster_version` |
| EKS module | `tf-script/infra/eks.tf` → `kubernetes_version` |
| Node pool values (non-prod) | `tf-script/infra/charts-values/node-pool/` |
| Node pool values (prod) | `tf-script/infra/prod-charts-values/node-pool/` |

| Rule | Detail |
|---|---|
| Order | Upgrade in AWS Console first, then update Terraform |
| Do not | Update Terraform before control plane upgrade completes |

---

## 4. Pre-Migration Activities

### 4.1 Upgrade EKS Add-ons (before control plane)

| Add-on | Action |
|---|---|
| CoreDNS | Update the version to the default recommended version |
| kube-proxy | Update the version to the default recommended version |
| Amazon VPC CNI | Update the version to the default recommended version |
| EBS CSI Driver | Update the version to the default recommended version (if used) |
| EFS CSI Driver | Update the version to the default recommended version (if used) |

After upgrade: confirm all add-ons are Active. Run `terraform plan` to check for drift.

### 4.2 Document Existing Node Groups / Node Pools

| Field to capture | Value |
|---|---|
| Name | |
| Kubernetes version | |
| AMI type (AL2 / AL2023 / Bottlerocket) | |
| Instance type | |
| Desired / min / max size | |
| Labels and taints | |
| Capacity type (On-Demand / Spot) | |
| Subnets, IAM role, disk size | |

---

## 5. Migration Steps

| Step | Action | Where | Done |
|---|---|---|---|
| 5.1 | Upgrade control plane to target version | AWS Console → EKS → Cluster → Upgrade | [ ] |
| 5.2 | Wait for upgrade to complete (~10–30 min) | AWS Console | [ ] |
| 5.3 | Validate cluster health | `kubectl get nodes` | [ ] |
| 5.4 | Upgrade add-ons to recommended versions | AWS Console → EKS → Add-ons | [ ] |
| 5.5 | Update `eks_cluster_version` in config YAML | `tf-script/configs/<environment>.yml` | [ ] |
| 5.6 | Run `terraform plan` and `apply` | Terminal / Github workflow | [ ] |

### 5.7 New Node Groups / Node Pools

| Phase | desired_size | min_size | max_size |
|---|---|---|---|
| Create (phase -1) | 0 | 0 | 1 |
| Scale up for migration | Same as old group | Same as desired | ≥ desired |
| Decommission old group | 0 | 0 | 1 |

| Setting | Value |
|---|---|
| AMI | Amazon Linux 2023 (AL2023) recommended |
| Wait for | All new nodes in Ready state |

Cluster Autoscaler tags (if used):

| Tag key | Tag value |
|---|---|
| `k8s.io/cluster-autoscaler/enabled` | `true` |
| `k8s.io/cluster-autoscaler/<cluster-name>` | `owned` |

### 5.8 Workload Migration

| # | Command / action |
|---|---|
| 1 | Scale down Cluster Autoscaler OR lock old ASG (min = max = desired) |
| 2 | `kubectl cordon <old-node>` |
| 3 | Rolling restart Deployments |
| 4 | `kubectl get pods -o wide --all-namespaces` — verify pods on new nodes |
| 5 | `kubectl drain <old-node> --ignore-daemonsets --delete-emptydir-data` |
| 6 | Repeat steps 2–5 for each old node |

| Validation | Status |
|---|---|
| No pods in Pending | [ ] |
| Health checks passing | [ ] |
| Traffic working | [ ] |
| Old nodes drained | [ ] |

### 5.9 Cluster Autoscaler Upgrade

| Step | Command |
|---|---|
| Check diff | `helm diff upgrade cluster-autoscaler autoscaler/cluster-autoscaler --namespace kube-system --version <chart-version> -f old-values.yaml` |
| Upgrade | `helm upgrade cluster-autoscaler autoscaler/cluster-autoscaler --namespace kube-system --version <chart-version> -f old-values.yaml` |
| Compatibility | https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler |

| Validation | Status |
|---|---|
| CAS pods healthy | [ ] |
| Logs show no errors | [ ] |
| Monitored for 1+ hour | [ ] |
| Scale event observed | [ ] |

---

## 6. Rollback

| Component | Can rollback? | How |
|---|---|---|
| Control plane | No | Fix forward or restore from backup cluster |
| Cluster Autoscaler | Yes | `helm rollback cluster-autoscaler <revision> -n kube-system` |
| Node groups | Yes | Re-scale old groups. Cordon and drain new nodes |

---

## 7. Post-Migration Checklist

| # | Check | Done |
|---|---|---|
| 1 | All workloads on new nodes | [ ] |
| 2 | No deprecated APIs (re-run Pluto) | [ ] |
| 3 | Cluster Autoscaler working | [ ] |
| 4 | Monitoring and logging OK | [ ] |
| 5 | PV bindings intact | [ ] |
| 6 | DaemonSets on new nodes | [ ] |
| 7 | `terraform plan` shows no drift | [ ] |
| 8 | Config YAML updated to target version | [ ] |

---

## 8. Useful Commands

| Purpose | Command |
|---|---|
| Cluster version | `aws eks describe-cluster --name <cluster-name> --region <region> --query 'cluster.version'` |
| Node versions | `kubectl get nodes -o custom-columns=NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion` |
| List add-ons | `aws eks list-addons --cluster-name <cluster-name> --region <region>` |
| Unhealthy pods | `kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded` |
| PDBs | `kubectl get pdb -A` |
| Deprecated APIs | `pluto detect-files -d <manifest-directory>` |

---

