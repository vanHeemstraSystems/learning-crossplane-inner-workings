# 03 — YAML Configuration & XRDs (Crossplane v2)

## The v2 YAML Stack

Crossplane v2 uses **three** primary YAML object types (down from four in v1):

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  1. XRD (CompositeResourceDefinition)          [cluster-scoped]  │
│     apiVersion: apiextensions.crossplane.io/v2                   │
│     spec.scope: Namespaced                                        │
│     "What API do you want to offer?"                             │
│     NO claimNames — v2 XRDs do not support Claims               │
│                                                                  │
│  2. Composition                                [cluster-scoped]  │
│     apiVersion: apiextensions.crossplane.io/v1                   │
│     mode: Pipeline only (native P&T removed)                     │
│     "How do you implement that API?"                             │
│                                                                  │
│  3. CompositeResource (XR)                   [NAMESPACE-scoped]  │
│     Developer creates this directly in their namespace           │
│     No separate Claim needed — this IS the developer's object    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

-----

## 1. CompositeResourceDefinition (XRD) — v2

**File: `xrd-example.yaml`**

```yaml
# KEY DIFFERENCES FROM v1:
#   apiVersion: apiextensions.crossplane.io/v2   ← v2 API version
#   spec.scope: Namespaced                        ← default in v2 (explicit is clearer)
#   NO claimNames field                           ← Claims not supported in v2

apiVersion: apiextensions.crossplane.io/v2
kind: CompositeResourceDefinition
metadata:
  name: databases.platform.mycompany.com    # <plural>.<group>
spec:
  scope: Namespaced                          # Namespaced (default) or Cluster

  group: platform.mycompany.com
  names:
    kind: Database                           # Kind developers use directly
    plural: databases
    singular: database

  # NO claimNames — not supported in v2
  # NO connectionSecretKeys — connection secrets are composed explicitly in v2

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
              required:
                - parameters
              properties:

                # User-facing parameters (developer writes these)
                parameters:
                  type: object
                  required:
                    - storageGB
                    - region
                  properties:
                    storageGB:
                      type: integer
                      description: "Storage size in GB"
                      minimum: 20
                      maximum: 16384
                    region:
                      type: string
                      description: "Azure region"
                      enum: [westeurope, northeurope, eastus]
                    size:
                      type: string
                      description: "Database tier"
                      default: small
                      enum: [small, medium, large]
                    version:
                      type: string
                      description: "PostgreSQL major version"
                      default: "15"
                      enum: ["13", "14", "15"]
                    highAvailability:
                      type: boolean
                      description: "Enable zone-redundant HA"
                      default: false

                # spec.crossplane is reserved for Crossplane machinery
                # (compositionRef, resourceRefs, etc.)
                # You do NOT need to define it — Crossplane manages it
```

**What happens when you apply this XRD?**

```
kubectl apply -f xrd-example.yaml
         │
         ▼
Crossplane XRD Controller:
  1. Validates XRD (no claimNames allowed in v2)
  2. Generates ONE new namespaced CRD:
     databases.platform.mycompany.com   (namespace-scoped)
  3. Creates RBAC ClusterRoles for the XR kind
  4. XRD.Status.Established = True

Compare to v1:
  v1 generated TWO CRDs (cluster-scoped XR + namespace-scoped Claim)
  v2 generates ONE namespaced CRD
```

-----

## 2. Composition — v2 (Functions Pipeline Only)

In v2, native patch-and-transform is removed. **All compositions use the Pipeline mode with Composition Functions.**

**File: `composition-example.yaml`**

```yaml
apiVersion: apiextensions.crossplane.io/v1    # Composition API version is still v1
kind: Composition
metadata:
  name: databases-azure-postgresql
  labels:
    provider: azure
    db: postgresql
spec:
  compositeTypeRef:
    apiVersion: platform.mycompany.com/v1alpha1
    kind: Database

  mode: Pipeline    # Only supported mode in v2 (Resources mode removed)

  pipeline:

    # Step 1: function-patch-and-transform creates composed resources
    - step: patch-and-transform
      functionRef:
        name: function-patch-and-transform    # must be installed as a Function package
      input:
        apiVersion: pt.fn.crossplane.io/v1beta1
        kind: Resources
        resources:

          # Composed Resource 1: Azure Resource Group
          - name: resource-group
            base:
              apiVersion: network.azure.upbound.io/v1beta1
              kind: ResourceGroup
              # NOTE: no namespace here — composed resources inherit the XR's namespace
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

          # Composed Resource 2: PostgreSQL Flexible Server
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
                fromFieldPath: spec.parameters.region
                toFieldPath: spec.forProvider.location
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.version
                toFieldPath: spec.forProvider.version

          # Composed Resource 3: Firewall Rule
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

          # Composed Resource 4: Connection Secret (explicit in v2 — no built-in support)
          # In v2 you compose the connection Secret as a regular Kubernetes resource
          - name: connection-secret
            base:
              apiVersion: v1
              kind: Secret
              metadata:
                name: db-connection
              # The secret will be created in the same namespace as the XR
              # A function like function-kcl or function-go-templating
              # can populate the secret data from MR status fields

    # Step 2: function-auto-ready marks the XR Ready when all composed resources are Ready
    - step: auto-ready
      functionRef:
        name: function-auto-ready
```

> **v2 Note on connection secrets:** `connectionSecretKeys` on the XRD and `writeConnectionSecretToRef` are removed in v2. To expose connection details to workloads, compose a `Secret` resource explicitly in the pipeline (as shown above). See the [connection details guide](https://docs.crossplane.io/latest/guides/connection-details-composition/).

-----

## 3. CompositeResource (XR) — What the Developer Writes in v2

**File: `xr-example.yaml`**

```yaml
# In v2, this is the ONLY object the developer creates.
# No Claim. No cluster-scoped twin.
# Apply with: kubectl apply -f xr-example.yaml -n team-alpha

apiVersion: platform.mycompany.com/v1alpha1
kind: Database
metadata:
  name: my-app-database
  namespace: team-alpha       # ← developer's namespace (namespaced-scoped XR)
spec:
  parameters:
    storageGB: 50
    region: westeurope
    size: small
    version: "15"
    highAvailability: false
  # spec.crossplane is auto-populated by Crossplane after creation:
  # crossplane:
  #   compositionRef:
  #     name: databases-azure-postgresql
  #   resourceRefs:
  #     - apiVersion: network.azure.upbound.io/v1beta1
  #       kind: ResourceGroup
  #       name: my-app-database-rg
  #       namespace: team-alpha
  #     - ...
```

-----

## v1 vs v2 YAML Comparison

```
v1 Developer Workflow:                    v2 Developer Workflow:
──────────────────────────────────        ──────────────────────────────────

1. Apply CLAIM (namespace-scoped):        1. Apply XR (namespace-scoped):
   apiVersion: platform.../v1alpha1          apiVersion: platform.../v1alpha1
   kind: Database                            kind: Database
   namespace: team-alpha                     namespace: team-alpha
   spec:                                     spec:
     parameters:                               parameters:
       storageGB: 50                             storageGB: 50
     writeConnectionSecretToRef:  ← GONE       # No connection secret ref needed
       name: my-db-conn                         # Compose a Secret explicitly

2. Crossplane creates XR internally:     2. Done. XR IS the developer object.
   kind: XDatabase                          No cluster-scoped twin created.
   (cluster-scoped, confusing)
```

-----

## Installing Required Composition Functions

Before applying Compositions, install the Function packages:

```bash
# function-patch-and-transform: the standard patching function
kubectl apply -f - <<EOF
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata:
  name: function-patch-and-transform
spec:
  package: xpkg.crossplane.io/crossplane-contrib/function-patch-and-transform:v0.8.2
EOF

# function-auto-ready: marks XR Ready when all composed resources are Ready
kubectl apply -f - <<EOF
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata:
  name: function-auto-ready
spec:
  package: xpkg.crossplane.io/crossplane-contrib/function-auto-ready:v0.4.1
EOF

# Verify both are healthy
kubectl get functions
```

-----

## The Patch-and-Transform System (via Function)

The same patching capabilities exist in v2, but they run inside `function-patch-and-transform`:

```
Patch Types:
  FromCompositeFieldPath   XR.spec field  →  composed resource field
  ToCompositeFieldPath     composed field →  XR.status field (write back)
  CombineFromComposite     multiple XR fields → one composed field

Transform Types:
  map      { small → B_Standard_B1ms, medium → GP_Standard_D2s_v3 }
  string   fmt: "%s-rg"  (format with prefix/suffix)
  math     Multiply: 1024  (convert GB to MB)
  convert  string → int, int → string, etc.
```

-----

## Observing v2 XR Status

```bash
# Check the XR (developer's view — everything in their namespace)
kubectl get database my-app-database -n team-alpha

# Describe for full status including spec.crossplane.resourceRefs
kubectl describe database my-app-database -n team-alpha

# See all composed resources in the same namespace
kubectl get managed -n team-alpha

# Events — all in the same namespace
kubectl get events -n team-alpha --sort-by='.lastTimestamp'
```

-----

## Next Steps

➡️ [04 — Go Internals: How Crossplane Providers Are Built](../04-go-internals/README.md)
