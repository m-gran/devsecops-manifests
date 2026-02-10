# Kyverno Controller

This directory contains the [Kyverno](https://kyverno.io/) controller installation for our Kubernetes cluster.

## 📂 Repository Structure

```text
platform/kyverno/
├── kustomization.yaml          # The entry point for ArgoCD
└── base/                       # The Helm Chart installation
    └── kustomization.yaml
```

## 🚀 How It Works

**Deployment**: ArgoCD points to `platform/kyverno`. It installs the Kyverno controller using the official Helm Chart (defined in `base/`).

**Policy Management**:
Policies are **NOT** managed in this directory. They have been moved to [platform/kyvernopolicies/README.md](../kyvernopolicies/README.md) to separate the Lifecycle of the Controller from the Lifecycle of the Policies.

## 🛠️ Upgrading Kyverno

To upgrade the Kyverno version:
1.  Navigate to `base/kustomization.yaml`.
2.  Update the `chart` version.
3.  Commit and Push. ArgoCD will sync the new version.
