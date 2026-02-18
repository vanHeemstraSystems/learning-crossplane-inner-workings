# 05 — Terraform Integration: Crossplane v2 vs. Terraform & Upjet

## Two Philosophies (Still True in v2)

```
TERRAFORM APPROACH
  Write HCL → terraform plan → terraform apply
  Push-based, separate state file, manual drift detection

CROSSPLANE v2 APPROACH
  kubectl apply xr.yaml -n my-namespace → controller watches → continuous reconcile
  Pull-based, state in etcd, automatic drift correction, namespace-native
```

The v2 upgrade makes Crossplane’s advantages **stronger**:

- Everything in one namespace → easier observability with standard `kubectl`
- No Claim proxy object → less confusion when debugging
- Compose Kubernetes resources alongside cloud MRs → full app + infra in one XR

-----

## Updated Comparison Table

|Feature                     |Crossplane v2                |Terraform                |
|----------------------------|-----------------------------|-------------------------|
|Control plane               |Kubernetes (namespace-first) |Separate                 |
|Developer object            |One namespaced XR            |HCL module call          |
|Continuous reconciliation   |✅ Yes (~10 min)              |❌ Manual `apply`         |
|Infrastructure API design   |✅ XRD (v2, namespaced)       |❌ No                     |
|Language                    |YAML + Go                    |HCL                      |
|Multi-tenancy               |✅ Native namespaces          |❌ Complex workspaces     |
|GitOps compatible           |✅ Native (ArgoCD, Flux)      |⚠️ With tooling           |
|State store                 |etcd (in cluster, namespaced)|tfstate files/remote     |
|Drift detection             |✅ Automatic                  |❌ Manual `terraform plan`|
|Compose Kubernetes resources|✅ Yes (v2 new)               |❌ No                     |
|Plan preview                |❌ No native preview          |✅ `terraform plan`       |

-----

## When To Use What

|Scenario                                      |Recommendation                             |
|----------------------------------------------|-------------------------------------------|
|Platform team building self-service infra APIs|Crossplane v2 ✅                            |
|Existing large Terraform codebase             |Keep Terraform; plan v2 migration          |
|GitOps with ArgoCD / Flux                     |Crossplane v2 ✅                            |
|Scripted one-off environments                 |Terraform                                  |
|Namespace-isolated multi-team infra           |Crossplane v2 ✅ (namespaces = isolation)   |
|App + infra in one declarative object         |Crossplane v2 ✅ (v2 “compose any resource”)|
|Detailed plan/preview before apply            |Terraform (`terraform plan`)               |
|Team only knows HCL                           |Terraform (or invest in training)          |

-----

## Upjet in the v2 World

Upjet generates namespaced MRs for v2-compatible providers:

```
Terraform Provider Schema
         │
         ▼ Upjet generates:
┌──────────────────────────────────────────────┐
│  Go structs                                  │
│   +kubebuilder:resource:scope=Namespaced     │  ← v2 default
│   +kubebuilder:resource:scope=Cluster        │  ← kept for backward compat
│                                              │
│  CRD YAMLs:                                  │
│   namespaced/network_virtualnetwork.yaml     │
│   cluster-scoped/network_virtualnetwork.yaml │
│                                              │
│  Observe/Create/Update/Delete impls          │
│  (wraps terraform-plugin-sdk, no tfstate)    │
└──────────────────────────────────────────────┘
```

**v2 provider status (as of early 2026):**

|Provider          |Namespaced MRs                            |
|------------------|------------------------------------------|
|provider-aws      |✅ Fully available                         |
|provider-azure    |🔄 In progress (cluster-scoped still works)|
|provider-gcp      |🔄 In progress                             |
|provider-helm     |🔄 In progress                             |
|provider-terraform|🔄 In progress                             |

For Azure specifically: cluster-scoped MRs continue to work unchanged as `LegacyCluster` support. New namespaced MR CRDs are added incrementally. Monitor the [Upbound provider-azure releases](https://github.com/upbound/provider-azure).

-----

## State Storage: Terraform vs Crossplane

### Terraform tfstate

```json
{
  "resources": [{
    "type": "azurerm_virtual_network",
    "instances": [{ "attributes": { "id": "...", "location": "westeurope" } }]
  }]
}
```

Stored externally (S3, Azure Blob, Terraform Cloud). Separate from your cluster.

### Crossplane v2 MR Status (in etcd, namespace-scoped)

```yaml
apiVersion: network.azure.upbound.io/v1beta1
kind: VirtualNetwork
metadata:
  name: my-vnet
  namespace: team-alpha          # ← namespaced in v2
status:
  atProvider:
    id: /subscriptions/.../virtualNetworks/my-vnet
    location: westeurope
    provisioningState: Succeeded
  conditions:
    - type: Ready
      status: "True"
    - type: Synced
      status: "True"
```

State is stored **inside Kubernetes**, tied to the same namespace as your XR and team resources.

-----

## Migrating From Terraform to Crossplane v2

```
Phase 1: Import existing Terraform resources into namespaced Crossplane MRs
───────────────────────────────────────────────────────────────────────────
1. Create namespace for the team: kubectl create namespace team-alpha
2. Create ManagedResource YAML with the correct namespace
3. Add annotation: crossplane.io/external-name: <azure-resource-id>
4. kubectl apply -f resource.yaml -n team-alpha
5. Crossplane Observe() detects it exists → marks Ready (no create needed)
6. terraform state rm <resource>

Phase 2: Wrap MRs in v2 Compositions
───────────────────────────────────────────────────────────────────────────
1. Design XRD with scope: Namespaced
2. Write Composition using function-patch-and-transform pipeline
3. Developer applies XR directly in their namespace (no Claim)

Phase 3: Decommission Terraform pipelines
───────────────────────────────────────────────────────────────────────────
1. Remove Terraform CI/CD jobs
2. Archive tfstate as audit trail
3. RBAC managed via Kubernetes Roles in each namespace
```

-----

## Running Terraform Inside Crossplane (provider-terraform)

If full migration isn’t possible yet, wrap existing Terraform modules as namespaced Crossplane Workspaces:

```yaml
# Namespaced Workspace (v2 style — MR in developer's namespace)
apiVersion: tf.upbound.io/v1beta1
kind: Workspace
metadata:
  name: my-terraform-module
  namespace: team-alpha          # ← namespaced in v2
spec:
  forProvider:
    source: Remote
    module: git::https://github.com/myorg/my-tf-module.git//terraform?ref=v1.2.0
    vars:
      - key: location
        value: westeurope
```

This lets you **wrap existing Terraform modules** as namespaced Crossplane Managed Resources — a pragmatic migration step.

-----

## Next Steps

➡️ [06 — Kubernetes CRDs: The Extension Mechanism](../06-kubernetes-crds/README.md)
