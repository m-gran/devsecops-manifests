# Kyverno Policy Management

This directory contains the [Kyverno](https://kyverno.io/) configuration for our Kubernetes cluster. It is separated from the business logic (`apps/`) to ensure a clean distinction between **Platform Rules** and **Application Workloads**.

## 📂 Repository Structure

```text
platform/kyverno/
├── kustomization.yaml          # The entry point for ArgoCD
├── base/                       # The Helm Chart installation
│   └── kustomization.yaml
└── policies/                   # The Policy Definitions
    ├── kustomization.yaml      # Lists active policies
    ├── disallow-root.yaml      # Actual Policy Rule
    ├── require-probes.yaml     # Actual Policy Rule
    └── tests/                  # Unit Tests for Policies
        ├── kyverno-test.yaml   # Test Assertion File
        ├── pass-root.yaml      # Fixture: Good Pod
        └── fail-root.yaml      # Fixture: Bad Pod
```

## 🚀 How It Works

**Deployment**: ArgoCD points to `platform/kyverno`. It installs the Kyverno controller (from `base/`) and applies the ValidatingPolicy CRs (from `policies/`).

**Enforcement**: Once deployed, Kyverno watches the entire cluster. Any new application deployed to `apps/` is automatically validated against these rules.

**CI/CD**:
*   If you edit a **Policy** here, the CI runs `kyverno test` (Unit Tests).
*   If a developer edits an **App**, the CI runs `kyverno apply` (Integration Scan) to ensure their app complies with these policies.

## 🛠️ Workflow: Adding a New Policy

### Step 1: Create the Policy

Create a new YAML file in `policies/`. We use the `ValidatingPolicy` (CEL) format.

**Example**: `policies/require-labels.yaml`

```yaml
apiVersion: policies.kyverno.io/v1
kind: ValidatingPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: Enforce
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        kinds: ["Pod"]
        operations: ["CREATE", "UPDATE"]
  validatingAdmissionPolicy:
    validations:
      - expression: "has(object.metadata.labels) && 'team' in object.metadata.labels"
        message: "All pods must have a 'team' label."
```

### Step 2: Register the Policy

You must add the filename to `policies/kustomization.yaml` so ArgoCD picks it up.

```yaml
resources:
  - disallow-root.yaml
  - require-labels.yaml  # <--- Added this
```

### Step 3: Add Unit Tests (Mandatory)

We strictly follow "Test-Driven Policy" practices. You must prove your policy works.

1.  Create a **Pass** fixture (a Pod that has the label).
2.  Create a **Fail** fixture (a Pod missing the label).
3.  Add an entry to `policies/tests/kyverno-test.yaml`.

```yaml
- policy: require-labels
  rule: check-labels
  resource: pass-label.yaml
  kind: Pod
  result: pass

- policy: require-labels
  rule: check-labels
  resource: fail-label.yaml
  kind: Pod
  result: fail
```

### Step 4: Verify Locally

Before pushing, run the tests locally to ensure your logic is sound.

```bash
# Must be run from the root of the repo
kyverno test ./platform/kyverno/policies/tests
```

## 🔍 Troubleshooting

**"No resources found"**
If you see this when getting policies, wait for ArgoCD to sync. If it persists, ensure the `validatingpolicies` CRD is installed on the cluster.

**CI Failure: "Validation error"**
If the CI fails on an Application PR, it means the application violates a policy.
1.  Read the CI log to see which rule failed.
2.  Update the application manifest in `apps/` to fix the violation.
3.  **Do not** edit the policy to make the test pass unless the policy itself is incorrect.
