# 08 — Providers: Architecture & Namespaced Managed Resources

## What Changed in v2 for Providers

|Topic            |v1                                 |v2                                               |
|-----------------|-----------------------------------|-------------------------------------------------|
|MR scope         |Cluster-scoped                     |**Namespace-scoped** (cluster-scoped is legacy)  |
|ControllerConfig |Supported                          |**Removed** — use `DeploymentRuntimeConfig`      |
|Provider watching|Cluster-wide cache, cluster objects|Cluster-wide cache, namespaced objects           |
|Selective install|All CRDs installed                 |`ManagedResourceDefinition` for selective install|

-----

## What Is a Provider? (Unchanged)

A **Provider** is a Crossplane package (OCI image) containing:

1. CRDs for all the cloud resources it manages (now namespace-scoped)
1. A controller pod that reconciles those CRDs via the cloud API
1. Authentication via `ProviderConfig`

-----

## Provider Installation (v2)

**File: `provider-azure.yaml`**

```yaml
# Install the Azure family provider (base — provides ProviderConfig CRD)
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-family-azure
spec:
  # Fully qualified image URL (--registry flag removed in v2)
  package: xpkg.upbound.io/upbound/provider-family-azure:v1.4.0
  runtimeConfigRef:
    name: provider-azure-config    # references DeploymentRuntimeConfig (not ControllerConfig)
---
# DeploymentRuntimeConfig (replaces ControllerConfig, which is removed in v2)
apiVersion: pkg.crossplane.io/v1beta1
kind: DeploymentRuntimeConfig
metadata:
  name: provider-azure-config
spec:
  deploymentTemplate:
    spec:
      selector: {}
      template:
        spec:
          containers:
            - name: package-runtime
              resources:
                requests:
                  memory: 256Mi
                  cpu: 100m
                limits:
                  memory: 1Gi
                  cpu: 500m
---
# Install the network sub-provider
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-azure-network
spec:
  package: xpkg.upbound.io/upbound/provider-azure-network:v1.4.0
---
# Configure credentials
apiVersion: azure.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: azure-creds
      key: credentials
```

> **Note on `ControllerConfig`:** In v2, `ControllerConfig` is completely removed. Migrating means replacing every `controllerConfigRef` with a `runtimeConfigRef` pointing to a `DeploymentRuntimeConfig` object.

-----

## Selective Provider Resource Installation (New in v2)

v2 introduces `ManagedResourceDefinition` to install **only the specific resource types you need**, reducing CRD sprawl:

```yaml
# Install only the VirtualNetwork and Subnet CRDs from provider-azure-network
# (instead of all 50+ network CRDs)
apiVersion: pkg.crossplane.io/v1alpha1
kind: ManagedResourceDefinition
metadata:
  name: enable-network-resources
spec:
  providerRef:
    name: provider-azure-network
  resources:
    - group: network.azure.upbound.io
      kind: VirtualNetwork
    - group: network.azure.upbound.io
      kind: Subnet
```

This is especially useful in large clusters where installing 700+ Azure CRDs causes significant API server overhead.

-----

## Provider Authentication Options (Unchanged)

### Option A: Service Principal (JSON in Secret)

```bash
az ad sp create-for-rbac --name crossplane-sp \
  --role Contributor \
  --scopes /subscriptions/$SUBSCRIPTION_ID \
  --sdk-auth > azure-creds.json

kubectl create secret generic azure-creds \
  --from-file=credentials=azure-creds.json \
  -n crossplane-system
```

```yaml
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: azure-creds
      key: credentials
```

### Option B: Workload Identity (AKS + Azure AD)

```yaml
spec:
  credentials:
    source: OIDCTokenFile
```

### Option C: System-Assigned Managed Identity

```yaml
spec:
  credentials:
    source: SystemAssignedMSI
```

-----

## Namespaced MRs and ProviderConfig

In v2, both the XR and the MRs live in the developer’s namespace. The `ProviderConfig` can be either namespace-scoped or cluster-scoped, depending on the provider’s implementation:

```yaml
# MR in team-alpha namespace, referencing a ProviderConfig
apiVersion: network.azure.upbound.io/v1beta1
kind: VirtualNetwork
metadata:
  name: my-vnet
  namespace: team-alpha              # ← namespaced in v2
spec:
  providerConfigRef:
    name: default                    # references ProviderConfig (cluster or namespace-scoped)
  forProvider:
    location: westeurope
    resourceGroupName: my-rg
    addressSpace: ["10.0.0.0/16"]
```

-----

## Provider Families (Unchanged Pattern, Updated for v2)

Large providers are split into sub-providers to reduce resource usage:

```
provider-family-azure        ← Core: ProviderConfig CRD + auth only
  ├── provider-azure-network       ← VNet, Subnet, NSG (now namespaced CRDs)
  ├── provider-azure-dbforpostgresql  ← PostgreSQL resources
  ├── provider-azure-storage       ← Storage accounts
  ├── provider-azure-containerservice ← AKS
  └── ... (60+ sub-providers, each with namespaced CRDs in v2)
```

-----

## Provider Pod Architecture (Unchanged)

```
provider-azure Pod (crossplane-system)
├── main()
│    ├── controller-runtime Manager
│    ├── Register controllers (one per resource type)
│    └── Manager.Start() → goroutines

├── ResourceGroupController goroutine
│    ├── Watch: ResourceGroup in ALL namespaces   ← ClusterRole watches all namespaces
│    ├── Queue: work items when objects change
│    └── Reconcile loop:
│         ├── Get ResourceGroup (includes namespace)
│         ├── external.Observe() → call Azure API
│         ├── external.Create/Update() if needed
│         └── Update Status, Requeue

└── VirtualNetworkController goroutine
     └── (same pattern, but for namespace-scoped VNet objects)
```

-----

## Checking Provider Health in v2

```bash
# List installed providers
kubectl get providers

# Check DeploymentRuntimeConfig (replaces ControllerConfig)
kubectl get deploymentruntimeconfigs

# Check ManagedResourceDefinitions (selective install, v2 new)
kubectl get managedresourcedefinitions

# Check all namespaced managed resources
kubectl get managed -n team-alpha

# Check managed resources across all namespaces
kubectl get managed --all-namespaces

# Provider pod logs
kubectl logs -n crossplane-system \
  $(kubectl get pods -n crossplane-system \
    -l pkg.crossplane.io/revision=provider-azure-network \
    -o name) --tail=50
```

-----

## Next Steps

➡️ [09 — Hands-On Labs](../09-hands-on/README.md)
