---
date: 2026-05-26T14:01:20Z
researcher: Spruce
git_commit: e51d6081d527fc27a6fa1e26e43dc883e918a960
branch: main
repository: devsecops-infra
topic: "Align devsecops-manifests repo with cost-trimmed devsecops-infra (no LB Controller, NodePort exposure)"
tags: [handoff, manifest-repo, eks, nodeport, cost, scenarios, argocd]
status: complete
last_updated: 2026-05-26
last_updated_by: Spruce
type: implementation_strategy
---

# Handoff: align manifest repo with cost-trimmed infra

## Task(s)

Cost-trim pass on `devsecops-infra` is **done in-tree but not committed**. The companion repo `devsecops-manifests` (referenced as `git@github.com:m-gran/devsecops-manifests.git` in scenario break scripts) now contains Kubernetes manifests that are **incompatible** with the trimmed infra. Next agent must:

1. **Planned**: Update manifest repo so `demo-app-devsecops-lab-lb` Service no longer uses `type: LoadBalancer` (AWS LB Controller was removed from infra → NLB cannot be provisioned, Service will sit `Pending`).
2. **Planned**: Decide and align the NodePort number used for internet exposure with the EKS node SG rule (currently SG opens TCP **5000** in `main.tf:164-172`; scenario READMEs reference NodePort **30003** for client traffic).
3. **Planned**: Update scenario break/fix scripts in this repo (`scenarios/1-network-lockdown/`, `scenarios/2-service-selector-mismatch/`, `scenarios/4-dns-and-loadbalancer/`) to match whichever exposure decision lands.
4. **WIP / awaiting commit**: Uncommitted infra changes (see *Recent changes*). Commit after manifest-repo alignment is decided so the two repos move together.

User intent locked in current session:
- Internet exposure path = **NodePort + public node IP, no NLB** (saves ~$16/mo + LCU).
- `flows.md` (stale TGW/Security-VPC/Client-VPC topology) deleted — do not reintroduce.

## Critical References

- `main.tf:164-172` — current SG rule opens TCP 5000 to `var.allowed_cidrs`. This is the only externally-accessible port today. If manifest NodePort != 5000 (which it isn't — scenarios use 30003), exposure is broken until SG is changed or NodePort matches.
- `scenarios/2-service-selector-mismatch/break.sh:9` — manifest repo URL: `git@github.com:m-gran/devsecops-manifests.git`, path `apps/dev-team-a/devsecops-demo-app`.
- `scenarios/4-dns-and-loadbalancer/break.sh:9,73-83` — break script sed-patches `loadbalancer.yaml` for `type: LoadBalancer` and NLB annotations. Will break if file no longer has those strings.

## Recent changes

Uncommitted on `main`:

- `flows.md` — **deleted** (was stale TGW/firewall multi-VPC doc; architecture not deployed).
- `main.tf:151` — `min_size = 1` → `min_size = 0` (enables true scale-to-zero pause).
- `main.tf:153` — `desired_size = 2` → `desired_size = var.desired_nodes` (wires the previously-dead `variables.tf:37-41` variable; `eks-schedule.yml` and `eks-safety-plan.yaml` `-var=desired_nodes=…` now actually work).
- `main.tf` — removed `module.aws_lb_controller_irsa` (was at old lines 178-192) and `helm_release.aws_lb_controller` (old lines 195-233). No NLB provisioner in cluster now.
- `main.tf:177-182` — `time_sleep.wait_for_load_balancer_cleanup` renamed to `time_sleep.wait_for_eks_ready`; `destroy_duration = "3m"` → `"30s"` (no LB cleanup to wait on).
- `main.tf:195` — ArgoCD `helm_release` `depends_on` updated to the renamed `time_sleep`.
- `main.tf:216-228` — dropped `eks_public_route_table_ids` output; descriptions for `eks_vpc_id` / `eks_public_subnets` no longer mention TGW.

`terraform fmt` / `validate` not run (no terraform CLI in shell). Next agent should run before committing.

## Learnings

- **Manifest repo and infra repo are tightly coupled via scenario scripts.** Scenarios 2 & 4 `git clone` the manifest repo and `sed`-patch specific files (`apps/dev-team-a/devsecops-demo-app/base/service.yaml`, `.../base/loadbalancer.yaml`). Any rename or structural change there silently breaks `break.sh`.
- **Three label/selector contracts must stay consistent** across the manifest repo (visible in scenario 2 README and break.sh):
  - Service selector: `app.kubernetes.io/instance: dev-team-a`
  - LoadBalancer selector: `app.kubernetes.io/name: devsecops-lab-app`
  - Service `targetPort: http` (port 5000 on the pod)
- **Default app namespace** is `dev-team-a-dev` (envvar `APP_NAMESPACE` in every `break.sh`). Manifests must create/use this namespace.
- **ArgoCD is installed by infra** (`main.tf:187-211`, chart `argo-cd` v5.51.6, namespace `argocd`, server svc `NodePort`, admin password bcrypt-hashed via `var.argocd_admin_password_hash`). The manifest repo is what ArgoCD points at — confirm there's an `Application` CR somewhere pointing to `apps/dev-team-a/devsecops-demo-app` or an Argo `ApplicationSet`. **Not in infra repo**; must live in manifest repo or be applied out-of-band.
- **AWS LB Controller is gone.** Any Service `type: LoadBalancer` in the manifest repo will stay `<pending>` forever. Scenarios 2 & 4 currently assume an NLB at port 8080 — they will not behave as documented until manifests switch to NodePort.
- **NodePort/exposure mismatch is the biggest open risk.** SG opens 5000 (pod port). For NodePort access the SG must open the NodePort number (default range 30000-32767, scenarios reference 30003). Two valid fixes:
  - Change SG rule to TCP 30003 (or open the full NodePort range) — then update scenarios 1 & 4 break.sh that filter by `FromPort==5000`.
  - Keep SG on 5000 and use `hostPort: 5000` on the pod spec (bypasses NodePort) — simpler for lab but couples pod scheduling to node port availability.
- **`force_delete = false` on ECR** (`permanent.tf:6`) — repo survives `terraform destroy`. Image storage cost persists. Acceptable; flag if cost still creeps.

## Artifacts

- `main.tf` — trimmed infra (uncommitted).
- `variables.tf:37-41` — `desired_nodes` variable, now actually wired.
- `permanent.tf` — ECR repo (unchanged, survives destroy).
- `outputs.tf` — root outputs (unchanged).
- `providers.tf` — S3 backend `bucket-devsecops-infra`, key `infra/terraform.tfstate`, region `eu-north-1`.
- `.github/workflows/eks-lifecycle.yaml` — deploy/destroy workflow (unchanged).
- `.github/workflows/eks-schedule.yml` — scale workflow (now actually scales due to `desired_nodes` wiring).
- `.github/workflows/eks-safety-plan.yaml` — plan-only workflow.
- `scenarios/{1..5}/{README.md,break.sh,fix.sh}` — 5 break/fix scenarios. Scenario 2 & 4 break against new infra until manifest repo is aligned.
- `README.md:9` — claims EKS v1.29; actual default is v1.33 (`variables.tf:13-17`). Cosmetic drift, fix opportunistically.

## Action Items & Next Steps

For next agent in `devsecops-manifests` repo:

1. **Read manifest layout** at `apps/dev-team-a/devsecops-demo-app/base/` — expect `service.yaml`, `loadbalancer.yaml`, `deployment.yaml`. Confirm structure before editing.
2. **Swap `loadbalancer.yaml`** Service from `type: LoadBalancer` to `type: NodePort` (or delete the file entirely if redundant with `service.yaml`). Remove `service.beta.kubernetes.io/aws-load-balancer-*` annotations.
3. **Pin the NodePort** to a known value (recommend `30003` to match existing scenario README text). Set on the surviving Service.
4. **Coordinate SG**: either
   - bump `main.tf:164-172` SG rule to TCP 30003 (and update `scenarios/1/break.sh:37`, `scenarios/4/break.sh:42` `FromPort==5000` filters to `30003`), **or**
   - add `hostPort: 5000` on the pod container in `deployment.yaml` and leave SG alone.
5. **Update scenario docs** that reference port 8080 NLB (scenarios 1, 2, 4) to point at `node-public-ip:<NodePort>` instead.
6. **Re-run scenarios 2 & 4** end-to-end after manifest update — sed patterns must still match.
7. **Confirm ArgoCD App/ApplicationSet** lives somewhere reachable from the freshly-deployed cluster. If not, add it to the manifest repo so ArgoCD actually syncs the demo app.
8. **Commit infra changes** in `devsecops-infra` (`git add -u && git commit`) once manifest decisions land. Run `terraform fmt` + `terraform validate` first.
9. Optional: tighten `README.md:9` EKS version (v1.29 → v1.33).

## Other Notes

- Repo defaults (`variables.tf`): region `eu-north-1`, cluster `devsecops-lab`, k8s `1.33`, VPC `10.0.0.0/16`, subnets `10.0.1.0/24` & `10.0.2.0/24`, node types `t3.medium`/`t3.small` SPOT.
- `var.allowed_cidrs` default is `["8.8.8.8/32"]` — placeholder; user must set their public IP before deploy (`curl ifconfig.me`).
- AI agent IAM user (`aws_iam_user.ai_agent`, `main.tf:7-59`) has VPC Reachability Analyzer + EC2/EKS describe perms. Persists across destroys (workflow only targets `module.eks`, `module.vpc`, `helm_release.argocd` — see `eks-lifecycle.yaml:39-43`).
- Argo admin password default hash in `variables.tf:50` corresponds to `password123`. ArgoCD server exposed via NodePort only; access via `kubectl port-forward service/argocd-server -n argocd 8080:443` (see `outputs.tf:33-36`).
- No `thoughts/` directory existed before this handoff; created `thoughts/shared/handoffs/general/`. No `humanlayer` CLI present — sync step skipped.
- The five scenario `break.sh`/`fix.sh` scripts are **destructive against live AWS + manifest repo**. They assume cluster + manifest repo are already up and matching. Run with care.
