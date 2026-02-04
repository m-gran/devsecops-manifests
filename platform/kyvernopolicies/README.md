# Kyverno Policy Management

This directory contains the **Policy Definitions** for our Kubernetes cluster.
It is separated from the controller installation (`../kyverno/`) to allow for independent lifecycles.

## 📂 Repository Structure

```text
platform/kyvernopolicies/
├── base/                       # The Policy Definitions (formerly policies/)
│   ├── kustomization.yaml      # Lists active policies
│   ├── disallow-root.yaml      # Actual Policy Rule
│   └── require-labels.yaml     # Actual Policy Rule
└── tests/                      # Unit Tests for Policies
    ├── disallow-root.yaml/     # Test Suite for disallow-root
    │   ├── kyverno-test.yaml   # Test Assertion File
    │   ├── pass.yaml           # Fixture: Good Pod
    │   └── fail.yaml           # Fixture: Bad Pod
    └── require-labels/         # Test Suite for require-labels
```

## 🚀 How It Works

**Deployment**: ArgoCD points to `platform/kyvernopolicies`. It applies the `ValidatingPolicy` CRs found in `base/`.

**Enforcement**: Any new application deployed to `apps/` is automatically validated against these rules.

**CI/CD**:
*   **Policy Changes**: If you edit a Policy here, the CI runs `kyverno test` (Unit Tests) via `.github/workflows/kyverno-unit-tests.yaml`.
*   **App Changes**: If a developer edits an App, the CI runs `kyverno apply` (Integration Scan) via `.github/workflows/kyverno-apps.yaml` to ensure compliance.

## 🛠️ Workflow: Adding a New Policy

### Step 1: Create the Policy

Create a new YAML file in `base/`. we use the `ValidatingPolicy` (CEL) format.

**Example**: `base/require-probes.yaml`

```yaml
apiVersion: policies.kyverno.io/v1
kind: ValidatingPolicy
metadata:
  name: require-probes
spec:
  validationFailureAction: Enforce
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        kinds: ["Pod"]
        operations: ["CREATE", "UPDATE"]
  validatingAdmissionPolicy:
    validations:
      - expression: "has(object.spec.containers) && object.spec.containers.all(c, has(c.livenessProbe) && has(c.readinessProbe))"
        message: "All containers must have liveness and readiness probes."
```

### Step 2: Register the Policy

You must add the filename to `base/kustomization.yaml` so ArgoCD picks it up.

```yaml
resources:
  - disallow-root.yaml
  - require-labels.yaml
  - require-probes.yaml  # <--- Added this
```

### Step 3: Add Unit Tests (Mandatory)

We strictly follow "Test-Driven Policy" practices.

1.  Create a new directory in `tests/` matching your policy name (e.g., `tests/require-probes/`).
2.  Create a **Pass** fixture (e.g., `tests/require-probes/pass.yaml`).
3.  Create a **Fail** fixture (e.g., `tests/require-probes/fail.yaml`).
4.  Create the assertion file `tests/require-probes/kyverno-test.yaml`.

```yaml
name: require-probes-test
policies:
  - ../../base/require-probes.yaml
resources:
  - pass.yaml
  - fail.yaml
results:
  - policy: require-probes
    rule: require-probes
    resource: pass.yaml
    kind: Pod
    result: pass
  - policy: require-probes
    rule: require-probes
    resource: fail.yaml
    kind: Pod
    result: fail
```

### Step 4: Verify Locally

Run the tests locally to ensure your logic is sound.

```bash
# Must be run from the root of the repo
kyverno test ./platform/kyvernopolicies/tests
```
