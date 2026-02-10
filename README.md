# DevSecOps ArgoCD Manifests

This repository contains the GitOps manifests for the DevSecOps platform, managed by ArgoCD.

to avoid loadbalancer costs in AWS we use NodePort instead of LoadBalancer

### Oneliner to access ArgoCD UI
kubectl port-forward service/argocd-server -n argocd 8080:443

## Directory Structure

*   **`bootstrap/`**: Contains the entry points for ArgoCD.
    *   **`root-app.yaml`**: The root Application that bootstraps the Control Plane. Pointer: `bootstrap/control-plane`.
    *   **`control-plane/`**: "Meta-Apps" that manage the platform configuration.
        *   `projects.yaml`: Loads ArgoCD Projects from `projects/`.
        *   `appsets.yaml`: Loads ApplicationSets from `clusters/`.
*   **`clusters/`**: Contains ApplicationSets. Each file here represents a Team or a Category of applications (e.g., `operations.yaml`, `dev-team-a.yaml`).
*   **`projects/`**: ArgoCD Project definitions (e.g., `operations-project.yaml`).
*   **`apps/`**: The actual application manifests. Organized by team/tenant.
    *   **`operations/`**: Operational tools (e.g., Kyverno).
    *   **`dev-team-a/`**: Applications belonging to Dev Team A.

## Workflows

### Onboarding a New Team
1.  Create a Project file in `projects/<team-name>-project.yaml`.
2.  Create an ApplicationSet in `clusters/<team-name>.yaml` targeting `apps/<team-name>/*`.

### Adding a New Application
1.  Create a new directory in `apps/<team-name>/<app-name>`.
2.  Add a `base/kustomization.yaml` (and overlays if needed).
3.  The ApplicationSet in `clusters/` will automatically detect and deploy it.

### Operations Apps
Operations applications (like Kyverno) are located in `apps/operations`. They are automatically deployed by the `clusters/operations.yaml` ApplicationSet.