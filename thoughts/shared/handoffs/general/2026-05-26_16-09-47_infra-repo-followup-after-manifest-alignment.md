---
date: 2026-05-26T14:09:47Z
researcher: Spruce
git_commit: 1ea99da75ae01e4f6e2c603298ccc18b7904eaee
branch: main
repository: devsecops-manifests
topic: "Infra-repo follow-up after manifest alignment (SG port + scenario rewrites + commit trimmed infra)"
tags: [handoff, devsecops-infra, eks, security-group, scenarios, nodeport]
status: open
last_updated: 2026-05-26
last_updated_by: Spruce
type: implementation_strategy
---

# Handoff: devsecops-infra follow-up after manifest alignment

## Task(s)

Manifest repo (`devsecops-manifests`, commit `1ea99da`) was aligned to the cost-trimmed
infra: no AWS LB Controller, NodePort-only exposure. Next agent works in
**`devsecops-infra`** (the trimmed-but-uncommitted in-tree changes from the prior
handoff `2026-05-26_16-01-20_align-manifest-repo-with-infra.md` still apply — commit
them as part of this work).

1. **Planned**: Bump EKS node SG ingress rule at `main.tf:164-172` from TCP **5000**
   to TCP **30003** (matches the dev-overlay nodePort the demo app now pins). Keep
   `var.allowed_cidrs` source unchanged.
2. **Planned**: Update scenario break/fix scripts that hard-code `FromPort==5000`:
   - `scenarios/1-network-lockdown/break.sh:37`
   - `scenarios/4-dns-and-loadbalancer/break.sh:42`
   Change the filter to `FromPort==30003` (or use the rule description as the
   selector — more robust to future port changes).
3. **Planned**: Rewrite `scenarios/2-service-selector-mismatch/break.sh` —
   `loadbalancer.yaml` no longer exists in manifests; the script must now sed-patch
   `apps/dev-team-a/devsecops-demo-app/base/service.yaml` instead. Update the
   selector typo target (was the typo `devsecops-lab-appp` in `loadbalancer.yaml`;
   reintroduce as a typo in `service.yaml` selector during break, revert in fix).
4. **Planned**: Rewrite or retire `scenarios/4-dns-and-loadbalancer/`. No NLB
   exists anymore; the "broken LB annotation / SG-on-LB-eni" framing is gone.
   Options:
   - Repurpose as DNS-only scenario (CoreDNS misconfig).
   - Convert to "NodePort SG drift" (close 30003 in SG, validate connection
     failure, fix by re-opening).
   - Drop scenario 4 entirely.
5. **Planned**: Run `terraform fmt` + `terraform validate` on the trimmed
   `main.tf`, then commit the previously uncommitted infra changes (see *Recent
   changes* in the prior handoff).
6. **Optional**: `README.md:9` claims EKS v1.29; actual default is v1.33
   (`variables.tf:13-17`). Fix opportunistically.

## Critical References

- `devsecops-infra/main.tf:164-172` — SG ingress rule, currently TCP 5000.
- `devsecops-infra/scenarios/1-network-lockdown/break.sh:37` — filters SG rules by `FromPort==5000`.
- `devsecops-infra/scenarios/4-dns-and-loadbalancer/break.sh:42,73-83` — filters by port 5000 and sed-patches `loadbalancer.yaml`.
- `devsecops-infra/scenarios/2-service-selector-mismatch/break.sh:9` — clones manifest repo, expects `apps/dev-team-a/devsecops-demo-app/base/loadbalancer.yaml`. **File no longer exists** in manifests.
- `devsecops-manifests` commit `1ea99da` — manifest alignment commit; see message body for the contract.

## Recent changes (in manifest repo, this session)

Committed on `main` of `devsecops-manifests` as `1ea99da align demo-app with NodePort-only exposure`:

- **Deleted** `apps/dev-team-a/devsecops-demo-app/base/loadbalancer.yaml` (was the broken `type: LoadBalancer` + NLB annotations Service, with a deliberate selector typo `devsecops-lab-appp`).
- **Deleted** `lb-test-aws.md` (root-level scratch `type: LoadBalancer` Service in `dev-team-a-dev`).
- **Removed** `loadbalancer.yaml` entry from `apps/dev-team-a/devsecops-demo-app/base/kustomization.yaml`.
- **Flipped** `apps/dev-team-a/devsecops-demo-app/base/deployment.yaml:33` `containerPort: 4999 → 5000` so the named `http` port aligns with the Service `targetPort: http`.

Untouched but relevant:
- `apps/.../base/service.yaml` — `type: NodePort`, port 5000, targetPort `http`, correct selectors.
- `apps/.../overlays/dev/service-patch.yaml` — pins `nodePort: 30003`.
- `apps/.../overlays/prod/service-patch.yaml` — pins `nodePort: 30001`.
- ArgoCD App-of-Apps (`bootstrap/root-app.yaml` → `clusters/dev-team-a.yaml` ApplicationSet) wired to the manifest repo; syncs both overlays into `dev-team-a-dev` and `dev-team-a-prod` namespaces with `CreateNamespace=true`.

## Resulting traffic path

```
internet → node public IP : 30003 (NodePort)
        → svc demo-app-devsecops-lab-app : 5000
        → pod devsecops-lab-app : 5000 (named "http")
```

Today this path is **closed at the SG layer** — SG only allows TCP 5000.
Task 1 above (SG bump to 30003) unblocks it.

If prod (`nodePort: 30001`) also needs internet exposure, open 30001 too — but
prod isn't deployed in the typical lab cycle, so deferring is fine.

## Learnings

- **Manifest repo / infra repo contract is now NodePort-only**, with port numbers
  pinned per overlay (dev=30003, prod=30001). Any SG change in infra must mirror
  these numbers, and any nodePort change in manifests must mirror the SG.
- **Selector contracts in the manifest base** (verified post-cleanup):
  - Service + Deployment selector: `app.kubernetes.io/name: devsecops-lab-app` + `app.kubernetes.io/instance: dev-team-a`
  - Service `targetPort: http`, Deployment named port `http: 5000`
- **Scenario 2's break has lost its target.** The selector typo previously lived in `loadbalancer.yaml`; now it has to be reintroduced in `service.yaml` during break and reverted during fix. The fix step also needs to confirm there's no leftover Service after rebase.
- **Scenario 4's whole premise is gone.** Without an NLB / LB Controller there's no LB to break. Treat as new scenario design, not a sed-patch update.
- **`force_delete = false` on ECR** (`permanent.tf:6`) — same caveat as prior handoff; image storage cost persists across `terraform destroy`. Acceptable.

## Artifacts

- `devsecops-manifests` HEAD `1ea99da` — current manifest state (post-alignment).
- `devsecops-manifests/apps/dev-team-a/devsecops-demo-app/base/{deployment,service,kustomization}.yaml` — authoritative spec for what infra must accommodate.
- `devsecops-manifests/apps/dev-team-a/devsecops-demo-app/overlays/{dev,prod}/service-patch.yaml` — nodePort pins.
- Prior handoff: `thoughts/shared/handoffs/general/2026-05-26_16-01-20_align-manifest-repo-with-infra.md` (in this repo) — full context on the cost-trim pass and uncommitted infra diffs.

## Action Items & Next Steps

For next agent in `devsecops-infra`:

1. **SG port bump** — `main.tf:164-172`, TCP 5000 → 30003. Keep `var.allowed_cidrs` semantics. Consider adding a `description` so future scripts can filter on description, not port.
2. **Patch scenario scripts** — `scenarios/1-network-lockdown/break.sh` and `scenarios/4-dns-and-loadbalancer/break.sh` `FromPort==5000` → `FromPort==30003` (or by description).
3. **Rewrite `scenarios/2-service-selector-mismatch/break.sh`** to sed-patch `apps/dev-team-a/devsecops-demo-app/base/service.yaml` instead of `loadbalancer.yaml`. Pick a distinctive break (e.g., flip selector value `devsecops-lab-app` → `devsecops-lab-appp`). Mirror the change in `fix.sh`.
4. **Decide scenario 4 fate** — rewrite as DNS-only / SG-drift, or remove. README + break.sh + fix.sh all need updating either way.
5. **Run `terraform fmt && terraform validate`**, then `git add -u && git commit` the trimmed infra (see prior handoff for the file list — `main.tf`, removed `flows.md`).
6. **Re-run scenarios 1, 2, 3, 5 end-to-end** against a freshly deployed cluster to confirm the SG/NodePort path works and break/fix scripts don't regress. Scenario 4 follows once decision in (4) is implemented.
7. **Optional**: README EKS version drift (1.29 → 1.33).

## Other Notes

- `var.allowed_cidrs` default `["8.8.8.8/32"]` — user must set their public IP before deploy.
- ArgoCD admin password still bcrypt-hashed `password123` (`variables.tf:50` default). ArgoCD server is NodePort; access via `kubectl port-forward service/argocd-server -n argocd 8080:443`.
- AI agent IAM user (`aws_iam_user.ai_agent`) persists across destroys — VPC Reachability Analyzer + EC2/EKS describe scoped.
- Manifest repo `thoughts/` is the cross-repo handoff store. Update `status:` to `complete` on the prior handoff once this one is closed.
