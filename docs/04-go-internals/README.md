# 04 — Go Internals: How Crossplane v2 Providers Are Built

## Why Go? (Unchanged from v1)

Crossplane and all its providers are written in **Go (Golang)** for the same reasons Kubernetes itself is:

- Compiles to a **single static binary** — ideal for containers
- **Strong concurrency** via goroutines — runs hundreds of controllers simultaneously
- The **Kubernetes controller-runtime** library is Go-native
- **`controller-gen`** generates CRD YAML from Go struct annotations

## What Changed in v2 for Go Providers

The core Go patterns are the same as v1 (the `ExternalClient` interface, the reconcile loop). The key v2 changes are:

- Managed Resource Go structs now use **namespaced scope** in their CRD registration
- Provider pods watch **namespaced** MR objects rather than cluster-scoped ones
- `ControllerConfig` type is removed — use `DeploymentRuntimeConfig`
- New providers publish **both** namespaced and cluster-scoped CRD variants for backward compatibility

-----

## Key Go Libraries

```
crossplane/crossplane-runtime     → base types, reconciler helpers
kubernetes-sigs/controller-runtime → controller scaffolding, watches, caches
upbound/upjet                     → Terraform-to-Crossplane code generator
k8s.io/apimachinery               → Kubernetes types (ObjectMeta, GVK, etc.)
k8s.io/client-go                  → Kubernetes API client
```

-----

## A Namespaced Managed Resource in Go

In v2, the CRD registration changes from `scope: Cluster` to `scope: Namespaced`:

```go
package v1beta1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    v1 "github.com/crossplane/crossplane-runtime/apis/common/v1"
)

// VirtualNetworkParameters are the configurable fields of a VirtualNetwork.
type VirtualNetworkParameters struct {
    // +kubebuilder:validation:Required
    Location string `json:"location"`

    // +kubebuilder:validation:Required
    ResourceGroupName string `json:"resourceGroupName,omitempty"`

    AddressSpace []string `json:"addressSpace"`
}

// VirtualNetworkObservation contains fields returned by the Azure API.
type VirtualNetworkObservation struct {
    ID                *string `json:"id,omitempty"`
    ProvisioningState *string `json:"provisioningState,omitempty"`
    Etag              *string `json:"etag,omitempty"`
}

// VirtualNetworkStatus defines the observed state.
type VirtualNetworkStatus struct {
    v1.ConditionedStatus `json:",inline"`
    AtProvider VirtualNetworkObservation `json:"atProvider,omitempty"`
}

// VirtualNetworkSpec defines the desired state.
type VirtualNetworkSpec struct {
    v1.ResourceSpec `json:",inline"`
    ForProvider VirtualNetworkParameters `json:"forProvider"`
}

// VirtualNetwork is the Schema for the VirtualNetworks API.
//
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
//
// KEY v2 CHANGE: scope is now Namespaced
// +kubebuilder:resource:scope=Namespaced,categories=crossplane;managed;azure
//
// +kubebuilder:printcolumn:name="READY",type="string",JSONPath=".status.conditions[?(@.type=='Ready')].status"
// +kubebuilder:printcolumn:name="SYNCED",type="string",JSONPath=".status.conditions[?(@.type=='Synced')].status"
type VirtualNetwork struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   VirtualNetworkSpec   `json:"spec"`
    Status VirtualNetworkStatus `json:"status,omitempty"`
}
```

The `+kubebuilder:resource:scope=Namespaced` marker (instead of `scope=Cluster`) is what causes `controller-gen` to produce a namespaced CRD:

```bash
# Generate CRDs — now produces namespaced scope
controller-gen crd:trivialVersions=true rbac:roleName=provider-azure \
  paths="./..." output:crd:artifacts:config=package/crds
```

-----

## The ExternalClient Interface (Unchanged)

Every provider still implements the same `ExternalClient` interface:

```go
type ExternalClient interface {
    Observe(ctx context.Context, mg resource.Managed) (ExternalObservation, error)
    Create(ctx context.Context, mg resource.Managed) (ExternalCreation, error)
    Update(ctx context.Context, mg resource.Managed) (ExternalUpdate, error)
    Delete(ctx context.Context, mg resource.Managed) error
}
```

### ExternalObservation — the reconciler decision object

```go
type ExternalObservation struct {
    ResourceExists          bool
    ResourceUpToDate        bool
    ResourceLateInitialized bool
    ConnectionDetails       managed.ConnectionDetails
    Annotations             map[string]string
}
```

-----

## The Reconciliation Decision Tree (Unchanged in v2)

```go
func (r *Reconciler) Reconcile(ctx context.Context, req reconcile.Request) (reconcile.Result, error) {

    // req.Namespace is now populated (namespaced MRs)
    mg := &v1beta1.VirtualNetwork{}
    r.client.Get(ctx, req.NamespacedName, mg)   // NamespacedName has both Name AND Namespace

    observation, err := r.external.Observe(ctx, mg)

    if !observation.ResourceExists {
        r.external.Create(ctx, mg)
        return reconcile.Result{RequeueAfter: 30 * time.Second}, nil
    }

    if !observation.ResourceUpToDate {
        r.external.Update(ctx, mg)
        return reconcile.Result{RequeueAfter: 30 * time.Second}, nil
    }

    mg.SetConditions(xpv1.Available())
    r.client.Status().Update(ctx, mg)

    return reconcile.Result{RequeueAfter: r.pollInterval}, nil
}
```

The key v2 difference: `req.NamespacedName` now includes the **namespace** of the MR (same namespace as the XR that composed it). The provider controller watches MRs in all namespaces using a cluster-scoped `Watch`, but the objects themselves are namespace-scoped.

-----

## DeploymentRuntimeConfig (Replaces ControllerConfig)

In v2, `ControllerConfig` is removed. Use `DeploymentRuntimeConfig` instead:

```go
// v1 (REMOVED in v2):
// apiVersion: pkg.crossplane.io/v1alpha1
// kind: ControllerConfig

// v2 replacement:
// apiVersion: pkg.crossplane.io/v1beta1
// kind: DeploymentRuntimeConfig
```

```yaml
# v2 DeploymentRuntimeConfig example
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
```

-----

## Upjet: Terraform → Crossplane Code Generator

**Upjet** generates Crossplane providers from Terraform provider schemas. In v2, Upjet-generated providers emit **both** namespaced and cluster-scoped CRD variants:

```
Terraform Provider Schema (JSON)
         │
         ▼ Upjet generates:
┌─────────────────────────────────────────────┐
│  Go types with +kubebuilder:resource:       │
│    scope=Namespaced   (v2 default)          │
│    scope=Cluster      (legacy, backward     │
│                        compatible)          │
│                                             │
│  CRD YAMLs — one set namespaced,            │
│              one set cluster-scoped         │
│                                             │
│  Observe/Create/Update/Delete wrapping      │
│  the Terraform resource functions           │
└─────────────────────────────────────────────┘
```

Provider releases that support v2 install namespaced CRDs by default. The cluster-scoped CRDs remain available for backward compatibility with v1-style workloads.

-----

## Provider Watching: Namespaced vs Cluster-Scoped

```go
// v1 provider controller watched cluster-scoped resources:
mgr.GetCache().Start(ctx)
ctrl.NewControllerManagedBy(mgr).
    For(&v1beta1.VirtualNetwork{}).   // cluster-scoped watch
    Complete(r)

// v2 provider controller watches namespaced resources
// (controller-runtime handles multi-namespace watching automatically)
ctrl.NewControllerManagedBy(mgr).
    For(&v1beta1.VirtualNetwork{}).   // namespace-scoped; mgr watches all namespaces
    Complete(r)
```

The controller-runtime `Manager` is started with a cluster-scoped cache that sees all namespaces. Individual MR objects are namespace-scoped, but the provider pod has cluster-wide watch permissions via its `ClusterRole`.

-----

## Next Steps

➡️ [05 — Terraform Integration & Upjet](../05-terraform-integration/README.md)
