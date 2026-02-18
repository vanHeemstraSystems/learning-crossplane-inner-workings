# 02 — Architecture: v2 Control Loop & Components

## The Kubernetes Reconciliation Model (Unchanged)

Crossplane inherits Kubernetes’ **declarative reconciliation model**:

```
You declare:    "I want an Azure PostgreSQL server in team-alpha namespace"
Crossplane:     checks Azure API → server doesn't exist
Provider:       calls Azure REST API to create it
               loops every ~10 minutes, correcting drift
```

What **changed in v2** is the *scope* of where objects live — everything is now namespaced.

-----

## v2 Component Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        KUBERNETES API SERVER + etcd                        │
│                                                                            │
│  CLUSTER-SCOPED objects:                                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐  │
│  │    XRDs     │  │ Compositions│  │  Providers   │  │  Functions     │  │
│  │ (generates  │  │ (blueprints)│  │  (packages)  │  │  (packages)    │  │
│  │ namespaced  │  │             │  │              │  │                │  │
│  │    CRDs)    │  │             │  │              │  │                │  │
│  └─────────────┘  └─────────────┘  └──────────────┘  └────────────────┘  │
│                                                                            │
│  NAMESPACE-SCOPED objects (e.g. team-alpha):                               │
│  ┌─────────────────┐  ┌──────────────────┐  ┌──────────────────────────┐  │
│  │ CompositeResource│  │ ManagedResources │  │ Secrets (connection info) │  │
│  │ (XR) — what the │  │ (ResourceGroup,  │  │ (composed explicitly)     │  │
│  │ developer creates│  │ FlexibleServer…) │  │                          │  │
│  └─────────────────┘  └──────────────────┘  └──────────────────────────┘  │
│                                                                            │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │              CROSSPLANE CORE POD (crossplane-system)                  │ │
│  │                                                                       │ │
│  │  ┌─────────────────┐  ┌──────────────────┐  ┌──────────────────────┐ │ │
│  │  │  XRD Controller  │  │  Composite       │  │  Package Manager     │ │ │
│  │  │  (generates      │  │  Reconciler      │  │  (installs Providers │ │ │
│  │  │   namespaced CRD)│  │  (runs Function  │  │   and Functions)     │ │ │
│  │  │                  │  │   pipeline)      │  │                      │ │ │
│  │  └─────────────────┘  └──────────────────┘  └──────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                            │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │              PROVIDER POD (crossplane-system)                         │ │
│  │  Watches namespaced MRs → calls cloud API → writes back Status       │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                            │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │              FUNCTION POD (crossplane-system)                         │ │
│  │  e.g. function-patch-and-transform                                    │ │
│  │  gRPC server called by Composite Reconciler                           │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────────┘
                                  │
                                  │ HTTPS / REST
                                  ▼
                       ┌─────────────────────┐
                       │   Azure Cloud API    │
                       └─────────────────────┘
```

-----

## The v2 Reconciliation Control Loop

```
USER: kubectl apply xr.yaml -n team-alpha
(creates Database XR directly — no Claim step)
         │
         ▼
┌────────────────────────────────────┐
│  Database XR lands in Kubernetes   │
│  namespace: team-alpha             │
│  (the developer's object directly) │
└─────────────────┬──────────────────┘
                  │
                  ▼
┌────────────────────────────────────┐
│  Composite Reconciler wakes up     │
│  - Reads XR.spec.crossplane.       │
│    compositionRef                  │
│  - Looks up matching Composition   │
└─────────────────┬──────────────────┘
                  │
                  ▼
┌────────────────────────────────────┐
│  Function Pipeline executes        │
│  (gRPC calls to Function pods)     │
│                                    │
│  Step 1: function-patch-and-       │
│          transform                 │
│    → returns desired MR manifests  │
│                                    │
│  Step 2: function-auto-ready       │
│    → marks XR Ready when all       │
│      composed resources are Ready  │
└─────────────────┬──────────────────┘
                  │
                  ▼
┌────────────────────────────────────┐
│  Crossplane applies desired MRs    │
│  IN THE SAME NAMESPACE (team-alpha)│
│  - ResourceGroup                   │
│  - FlexibleServer                  │
│  - Secret (connection details)     │
└─────────────────┬──────────────────┘
                  │
                  ▼
┌────────────────────────────────────┐
│  Provider pod reconciles each MR   │
│  - Calls Azure REST API            │
│  - Writes MR.Status.AtProvider     │
└─────────────────┬──────────────────┘
                  │
                  ▼
┌────────────────────────────────────┐
│  Status propagates up:             │
│  MR.Ready → XR.Status.Ready        │
│  XR.spec.crossplane.resourceRefs   │
│  updated with composed resource    │
│  names                             │
└────────────────────────────────────┘
                  │
          ↺ Loop repeats every ~10min
            (drift detection & correction)
```

-----

## Key Difference: Where `spec.crossplane` Lives

In v1, Crossplane machinery fields were mixed into `spec`:

```yaml
# v1 XR (cluster-scoped — user never wrote this directly)
spec:
  compositionRef:
    name: my-composition
  resourceRefs:
    - apiVersion: azure.upbound.io/v1beta1
      kind: ResourceGroup
      name: my-rg
  parameters:            # ← user fields and Crossplane fields mixed together
    storageGB: 50
```

In v2, everything Crossplane manages is cleanly isolated:

```yaml
# v2 XR (namespace-scoped — developer writes this directly)
spec:
  parameters:            # ← user fields only (clean for developers)
    storageGB: 50
    region: westeurope
  crossplane:            # ← all Crossplane machinery separated here
    compositionRef:
      name: databases-azure
    resourceRefs:
      - apiVersion: network.azure.upbound.io/v1beta1
        kind: ResourceGroup
        name: my-db-rg-x8k2j
        namespace: team-alpha
```

-----

## Namespace RBAC Isolation

v2’s namespace-first design enables **standard Kubernetes RBAC** to control infrastructure access:

```yaml
# Give team-alpha developers permission to create Database XRs
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: database-creator
  namespace: team-alpha
rules:
  - apiGroups: [platform.mycompany.com]
    resources: [databases]
    verbs: [get, list, watch, create, update, patch, delete]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-alpha-db-access
  namespace: team-alpha
subjects:
  - kind: Group
    name: team-alpha-developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: database-creator
  apiGroup: rbac.authorization.k8s.io
```

In v1, this required both a `Role` (for Claims) and a `ClusterRole` (for XRs). In v2, a single namespace `Role` is sufficient.

-----

## Composing Any Kubernetes Resource (v2)

v2 Compositions are no longer limited to Crossplane MRs. A single XR can compose:

```
XR (Database, namespace: team-alpha)
  │
  ├── MR: ResourceGroup          (Crossplane MR — Azure resource)
  ├── MR: FlexibleServer         (Crossplane MR — Azure resource)
  ├── Deployment                 (native Kubernetes — app server)
  ├── Service                    (native Kubernetes — load balancer)
  └── Secret                     (native Kubernetes — connection details)
```

> **Note:** Crossplane needs RBAC permissions to compose non-MR resources. See [Composition docs](https://docs.crossplane.io/latest/composition/compositions/#grant-access-to-composed-resources).

-----

## Operations (New in v2)

v2 introduces `Operations` for one-off and scheduled tasks:

```yaml
# Run a backup every night at 2 AM
apiVersion: ops.crossplane.io/v1alpha1
kind: CronOperation
metadata:
  name: nightly-db-backup
  namespace: team-alpha
spec:
  schedule: "0 2 * * *"
  mode: Pipeline
  pipeline:
    - step: trigger-backup
      functionRef:
        name: function-python
      # function reads the FlexibleServer MR and triggers a backup via Azure API
```

Operations support three modes: `Operation` (run once), `CronOperation` (scheduled), `WatchOperation` (trigger on resource change).

-----

## Next Steps

➡️ [03 — YAML Configuration & XRDs (v2)](../03-yaml-xrd/README.md)
