# 01 — Overview: Crossplane v2 — What Changed and Why

## The v1 Problem: Two Objects for One Job

In Crossplane v1, provisioning infrastructure required **two parallel objects** for every resource:

```
v1 Model (cluster-scoped XR + namespace-scoped Claim):

CLUSTER SCOPE                          NAMESPACE SCOPE
─────────────────────────────          ──────────────────────────────
CompositeResource (XR)                 Claim (XRC)
  kind: XDatabase                        kind: Database
  name: my-db-abc12   ◀──── proxy ────▶  name: my-db
  (Crossplane internal)                  namespace: team-alpha
                                         (developer writes this)
```

This dual-object model caused confusion:

- Why do two objects exist for the same thing?
- Which one do I `kubectl describe` when debugging?
- Status and events were split across both objects
- Cluster-scoped XRs meant developers couldn’t use standard `kubectl get all`

-----

## The v2 Solution: One Namespaced XR

Crossplane v2 **collapses the XR+Claim pair into a single namespaced XR**:

```
v2 Model (namespaced XR only):

TEAM-ALPHA NAMESPACE
─────────────────────────────────────────────────────
CompositeResource (XR)                    ← developer creates this directly
  kind: Database
  name: my-db
  namespace: team-alpha
  spec:
    parameters:
      storageGB: 50
    crossplane:                           ← Crossplane machinery lives here
      compositionRef:
        name: databases-azure
      resourceRefs:
        - kind: ResourceGroup
          name: my-db-rg-x8k2j
        - kind: FlexibleServer
          name: my-db-pg-m9n3p
```

The developer creates the XR directly in their namespace. No proxy Claim. No cluster-scoped twin.

-----

## What Crossplane v2 Removes

These v1 features are **gone** in `apiextensions.crossplane.io/v2`:

|Removed Feature                |v2 Replacement                                        |
|-------------------------------|------------------------------------------------------|
|`claimNames` in XRD            |No replacement — XRs are now namespaced directly      |
|`Claim` (XRC) objects          |Developers apply XRs directly                         |
|Built-in `connectionSecretKeys`|Compose a `Secret` resource explicitly                |
|Native patch-and-transform     |Composition Functions (`function-patch-and-transform`)|
|`ControllerConfig`             |`DeploymentRuntimeConfig`                             |
|External secret stores (alpha) |Removed; use composed Secrets                         |

-----

## What Crossplane v2 Adds

|New Feature                        |Description                                                          |
|-----------------------------------|---------------------------------------------------------------------|
|`scope: Namespaced` (default)      |XRs live in a namespace, composed resources in the same namespace    |
|`scope: Cluster`                   |Cluster-scoped XRs for platform-level resources (RBAC, global config)|
|`spec.crossplane.*` on XRs         |All Crossplane metadata cleanly isolated from user fields            |
|Compose **any** Kubernetes resource|Deployments, Services, ConfigMaps — not just MRs                     |
|`Operations` (alpha)               |CronOperation, WatchOperation for operational workflows              |
|Namespaced Managed Resources       |MRs live in the same namespace as the XR                             |
|`ManagedResourceDefinition`        |Selectively enable only the provider resources you need              |

-----

## The New Developer Experience

```
v1 Developer Workflow:
  1. Ask platform team "what Claim kind do I use?"
  2. Find the Claim CRD (namespace-scoped)
  3. kubectl apply claim.yaml -n my-namespace
  4. Wait for Claim to create XR (cluster-scoped)
  5. Debug issues across Claim AND XR AND MRs

v2 Developer Workflow:
  1. Ask platform team "what resource kind do I use?"
  2. kubectl apply xr.yaml -n my-namespace     ← one object
  3. Debug issues on the XR AND MRs — all in the same namespace
```

-----

## The New Platform Team Experience

```
v1 Platform Team Workflow:
  Write: XRD (cluster-scoped) + claimNames (for namespace proxy)
         Composition
  Manage: RBAC for XRs (cluster) + RBAC for Claims (namespace)

v2 Platform Team Workflow:
  Write: XRD (scope: Namespaced) — no claimNames
         Composition (Functions pipeline)
  Manage: RBAC for XRs in namespaces (standard Kubernetes patterns)
```

-----

## Backward Compatibility

Crossplane v2 is **backward compatible** with v1 workloads via `LegacyCluster` scope:

```yaml
# v1 XRDs continue to work unchanged
apiVersion: apiextensions.crossplane.io/v1    # ← still v1
kind: CompositeResourceDefinition
# ...
# scope defaults to LegacyCluster automatically
# Claims still work with LegacyCluster scope
```

```yaml
# v2 XRDs use the new API
apiVersion: apiextensions.crossplane.io/v2    # ← v2
kind: CompositeResourceDefinition
spec:
  scope: Namespaced                           # default in v2
  # claimNames NOT supported in v2
```

**The recommendation:** keep existing v1 workloads as-is, write all new workloads using v2 style.

-----

## Crossplane v2 in the Broader Landscape

```
Crossplane v2 scope expansion:

INFRASTRUCTURE ONLY (v1):               INFRASTRUCTURE + APPLICATIONS (v2):
────────────────────────────            ────────────────────────────────────
XR → MR → Cloud Resource               XR → MR → Cloud Resource
                                         XR → Deployment
                                         XR → Service
                                         XR → CloudNativePG Cluster
                                         XR → RBAC
                                         XR → CronOperation (backup schedule)
```

v2 allows platform teams to build **unified application + infrastructure** APIs. One XR can provision a database, deploy the application, configure networking, and set up monitoring — all within one namespace.

-----

## Next Steps

➡️ [02 — Architecture: v2 Control Loop & Components](../02-architecture/README.md)
