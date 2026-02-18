# 07 — Composite Resources: Namespaced XRs in Depth (v2)

## The v2 Abstraction Model

In v2, the XR IS the developer’s object. There is no intermediary Claim.

```
v1 ABSTRACTION LAYERS:                   v2 ABSTRACTION LAYERS:
────────────────────────────────         ────────────────────────────────
Platform team designs:                   Platform team designs:
  XRD (cluster-scoped)                     XRD (scope: Namespaced)
  Composition                              Composition (Functions pipeline)

Developer interacts with:                Developer interacts with:
  Claim (namespace-scoped proxy)     →     XR directly (namespace-scoped)

Crossplane creates internally:           Crossplane creates in same namespace:
  XR (cluster-scoped, hidden)              MR #1, MR #2, MR #3, Secret
  MR #1, MR #2, MR #3 (cluster-scoped)    (all namespace-scoped)
```

-----

## XR Scope in v2

```
scope: Namespaced (default)              scope: Cluster
─────────────────────────────────        ─────────────────────────────────────
XR lives in a namespace                  XR lives at cluster scope

Can compose:                             Can compose:
  - Any namespaced resource in           - Any cluster-scoped resource
    the SAME namespace                   - Any namespaced resource in
                                           ANY namespace

Best for:                                Best for:
  - Application databases                - Platform RBAC
  - Per-team networks                    - Cluster-wide config
  - Team-specific secrets                - Global DNS zones
  - Most platform APIs                   - Shared infrastructure
```

-----

## What the XR Object Looks Like After Creation

When a developer applies an XR, Crossplane populates `spec.crossplane.*`:

```yaml
# What the developer applied:
apiVersion: platform.mycompany.com/v1alpha1
kind: Database
metadata:
  name: my-app-database
  namespace: team-alpha
spec:
  parameters:
    storageGB: 50
    region: westeurope
    size: small

# What Crossplane adds (after first reconcile):
apiVersion: platform.mycompany.com/v1alpha1
kind: Database
metadata:
  name: my-app-database
  namespace: team-alpha
spec:
  parameters:
    storageGB: 50
    region: westeurope
    size: small
  crossplane:                            # ← Crossplane-managed section
    compositionRef:
      name: databases-azure-postgresql   # ← which Composition was selected
    compositionRevisionRef:
      name: databases-azure-postgresql-abc123
    resourceRefs:                        # ← all composed resources
      - apiVersion: network.azure.upbound.io/v1beta1
        kind: ResourceGroup
        name: my-app-database-rg
        namespace: team-alpha
      - apiVersion: dbforpostgresql.azure.upbound.io/v1beta2
        kind: FlexibleServer
        name: my-app-database-pg
        namespace: team-alpha
      - apiVersion: v1
        kind: Secret
        name: my-app-database-conn
        namespace: team-alpha
status:
  conditions:
    - type: Ready
      status: "True"
    - type: Synced
      status: "True"
```

-----

## Full v2 Azure Example

**File: `composite-azure.yaml`** — a complete working v2 example.

```yaml
# ─────────────────────────────────────────────────────────────────────
# 1/3: XRD — v2 style (namespaced, no claimNames)
# ─────────────────────────────────────────────────────────────────────
apiVersion: apiextensions.crossplane.io/v2
kind: CompositeResourceDefinition
metadata:
  name: databases.platform.mycompany.com
spec:
  scope: Namespaced           # default in v2; generates one namespaced CRD
  group: platform.mycompany.com
  names:
    kind: Database
    plural: databases
  # NO claimNames — not supported in v2
  # NO connectionSecretKeys — compose a Secret explicitly instead
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: [parameters]
              properties:
                parameters:
                  type: object
                  required: [region, storageGB]
                  properties:
                    region:
                      type: string
                      enum: [westeurope, northeurope, eastus]
                    storageGB:
                      type: integer
                      minimum: 20
                    size:
                      type: string
                      default: small
                      enum: [small, medium, large]
                    pgVersion:
                      type: string
                      default: "15"
                      enum: ["13","14","15"]
---
# ─────────────────────────────────────────────────────────────────────
# 2/3: Composition — Functions pipeline (only mode in v2)
# ─────────────────────────────────────────────────────────────────────
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: databases-azure-postgresql
  labels:
    provider: azure
spec:
  compositeTypeRef:
    apiVersion: platform.mycompany.com/v1alpha1
    kind: Database
  mode: Pipeline
  pipeline:
    - step: patch-and-transform
      functionRef:
        name: function-patch-and-transform
      input:
        apiVersion: pt.fn.crossplane.io/v1beta1
        kind: Resources
        resources:

          - name: resource-group
            base:
              apiVersion: network.azure.upbound.io/v1beta1
              kind: ResourceGroup
              spec:
                forProvider:
                  location: westeurope
            patches:
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.region
                toFieldPath: spec.forProvider.location
              - type: FromCompositeFieldPath
                fromFieldPath: metadata.name
                toFieldPath: metadata.name
                transforms:
                  - type: string
                    string:
                      fmt: "%s-rg"

          - name: postgresql-server
            base:
              apiVersion: dbforpostgresql.azure.upbound.io/v1beta2
              kind: FlexibleServer
              spec:
                forProvider:
                  location: westeurope
                  resourceGroupNameSelector:
                    matchControllerRef: true
                  administratorLogin: pgadmin
                  storageMb: 32768
                  skuName: B_Standard_B1ms
                  version: "15"
                  administratorPasswordSecretRef:
                    namespace: crossplane-system
                    name: pg-admin-password
                    key: password
            patches:
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.region
                toFieldPath: spec.forProvider.location
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.size
                toFieldPath: spec.forProvider.skuName
                transforms:
                  - type: map
                    map:
                      small:  B_Standard_B1ms
                      medium: GP_Standard_D2s_v3
                      large:  MO_Standard_E4s_v3
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.storageGB
                toFieldPath: spec.forProvider.storageMb
                transforms:
                  - type: math
                    math:
                      type: Multiply
                      multiply: 1024
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.pgVersion
                toFieldPath: spec.forProvider.version

          - name: firewall-allow-azure
            base:
              apiVersion: dbforpostgresql.azure.upbound.io/v1beta1
              kind: FlexibleServerFirewallRule
              spec:
                forProvider:
                  serverNameSelector:
                    matchControllerRef: true
                  resourceGroupNameSelector:
                    matchControllerRef: true
                  startIpAddress: 0.0.0.0
                  endIpAddress: 0.0.0.0

    - step: auto-ready
      functionRef:
        name: function-auto-ready
---
# ─────────────────────────────────────────────────────────────────────
# 3/3: XR — what the developer applies (no Claim needed)
# Apply with: kubectl apply -f this-file.yaml -n team-alpha
# ─────────────────────────────────────────────────────────────────────
apiVersion: platform.mycompany.com/v1alpha1
kind: Database
metadata:
  name: team-alpha-db
  namespace: team-alpha           # developer's namespace
spec:
  parameters:
    region: westeurope
    storageGB: 50
    size: small
    pgVersion: "15"
```

-----

## XR Lifecycle in v2

```
Status progression:

kubectl apply xr.yaml -n team-alpha
    │
    ▼
SYNCED=Unknown, READY=Unknown     ← XR just created, function not run yet
    │
    ▼
SYNCED=True, READY=False          ← MRs created in Azure, not provisioned yet
    │
    ▼
SYNCED=True, READY=True           ← All MRs Ready, XR is Ready

kubectl delete database team-alpha-db -n team-alpha
    │
    ▼
SYNCED=True, READY=False          ← Deletion propagating to MRs
    │
    ▼
(object removed from etcd)        ← All MRs deleted from Azure and cluster
```

```bash
# Watch the XR in real-time
kubectl get database team-alpha-db -n team-alpha -w

# NAME            SYNCED  READY  COMPOSITION                    AGE
# team-alpha-db   False   False                                 5s
# team-alpha-db   True    False  databases-azure-postgresql     15s
# team-alpha-db   True    True   databases-azure-postgresql     9m
```

-----

## Namespace Isolation in Practice

Each team namespace is a complete, isolated environment:

```bash
# Team Alpha's view — they only see their own resources
kubectl get all -n team-alpha
# Shows: Database XR, ResourceGroup MR, FlexibleServer MR, Secret, etc.

# Platform team's view — can see across namespaces
kubectl get databases --all-namespaces
# NAMESPACE     NAME           SYNCED  READY  AGE
# team-alpha    alpha-db       True    True   2d
# team-beta     beta-db        True    True   1d
# team-gamma    gamma-db       True    True   5h

# RBAC: team-alpha devs cannot see team-beta's resources
# (standard Kubernetes namespace isolation)
```

-----

## Next Steps

➡️ [08 — Providers: Architecture & Namespaced MRs](../08-providers/README.md)
