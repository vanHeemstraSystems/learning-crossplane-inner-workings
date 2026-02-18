# 06 — Kubernetes CRDs: The Extension Mechanism

## What Is a CRD?

A **CustomResourceDefinition (CRD)** is how Kubernetes allows you to extend its API. Crossplane uses CRDs heavily — both for Managed Resources (one CRD per cloud resource type) and for Composite Resources (one CRD per XRD).

In v2, **all these CRDs default to namespace-scoped** rather than cluster-scoped.

-----

## CRD Scope: The v1 → v2 Change

```
v1 Managed Resource CRD:                 v2 Managed Resource CRD:
────────────────────────────────         ──────────────────────────────────
scope: Cluster                           scope: Namespaced
(kubectl get virtualnetworks             (kubectl get virtualnetworks -n team-alpha
 shows cluster-wide objects)              shows only team-alpha's objects)

v1 XRD-generated CRDs:
  xdatabases.platform.com  → Cluster     v2 XRD-generated CRDs:
  databases.platform.com   → Namespaced  databases.platform.com → Namespaced
  (XR cluster, Claim namespace)          (XR IS namespaced — one CRD only)
```

-----

## Namespaced CRD Example

**File: `crd-example.yaml`** — a namespaced managed resource CRD (v2 style):

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: virtualnetworks.network.azure.upbound.io
  annotations:
    meta.crossplane.io/maintainer: Upbound

spec:
  group: network.azure.upbound.io
  names:
    kind: VirtualNetwork
    listKind: VirtualNetworkList
    plural: virtualnetworks
    singular: virtualnetwork
    categories:
      - crossplane
      - managed
      - azure

  # KEY v2 CHANGE: Namespaced (was Cluster in v1)
  scope: Namespaced

  versions:
    - name: v1beta1
      served: true
      storage: true
      subresources:
        status: {}
      additionalPrinterColumns:
        - name: READY
          type: string
          jsonPath: .status.conditions[?(@.type=='Ready')].status
        - name: SYNCED
          type: string
          jsonPath: .status.conditions[?(@.type=='Synced')].status
        - name: EXTERNAL-NAME
          type: string
          jsonPath: .metadata.annotations.crossplane\.io/external-name
        - name: AGE
          type: date
          jsonPath: .metadata.creationTimestamp

      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: [forProvider]
              properties:
                deletionPolicy:
                  type: string
                  default: Delete
                  enum: [Delete, Orphan]
                providerConfigRef:
                  type: object
                  default:
                    name: default
                  properties:
                    name:
                      type: string
                  required: [name]
                forProvider:
                  type: object
                  required: [location, resourceGroupName, addressSpace]
                  properties:
                    location:
                      type: string
                    resourceGroupName:
                      type: string
                    resourceGroupNameRef:
                      type: object
                      properties:
                        name:
                          type: string
                    addressSpace:
                      type: array
                      items:
                        type: string
                    dnsServers:
                      type: array
                      items:
                        type: string
                    tags:
                      type: object
                      additionalProperties:
                        type: string
            status:
              type: object
              properties:
                conditions:
                  type: array
                  items:
                    type: object
                    properties:
                      type: {type: string}
                      status: {type: string}
                      reason: {type: string}
                      message: {type: string}
                      lastTransitionTime: {type: string, format: date-time}
                atProvider:
                  type: object
                  properties:
                    id: {type: string}
                    etag: {type: string}
                    provisioningState: {type: string}
                    guid: {type: string}
```

-----

## How XRDs Generate CRDs in v2

```
XRD (apiextensions.crossplane.io/v2):
  spec.scope: Namespaced
  spec.names.kind: Database
         │
         ▼
XRD Controller generates:

  v1:                                   v2:
  TWO CRDs                              ONE CRD
  ──────────────────────────            ──────────────────────────
  xdatabases.platform...                databases.platform...
    scope: Cluster                        scope: Namespaced
  databases.platform...
    scope: Namespaced
  (Claim proxy)
```

No more cluster-scoped XR CRD. No more namespace-scoped Claim CRD. Just one namespaced CRD that the developer uses directly.

-----

## Useful CRD Commands in v2

```bash
# List all CRDs (both namespaced and cluster-scoped)
kubectl get crds

# Check the scope of a CRD
kubectl get crd virtualnetworks.network.azure.upbound.io \
  -o jsonpath='{.spec.scope}'
# → Namespaced

# Check the scope of the XRD-generated CRD
kubectl get crd databases.platform.mycompany.com \
  -o jsonpath='{.spec.scope}'
# → Namespaced   (v2 generates namespaced CRD)

# See all Database XRs across all namespaces
kubectl get databases --all-namespaces

# See only team-alpha's databases
kubectl get databases -n team-alpha

# See all managed resources in a namespace
kubectl get managed -n team-alpha

# Cross-namespace view of all managed resources
kubectl get managed --all-namespaces
```

-----

## CRD Validation (Unchanged)

Kubernetes validates objects against the CRD schema on admission:

```
kubectl apply -f xr.yaml -n team-alpha
         │
         ▼
kube-apiserver admission:
  ├── Required fields present?        → 400 if missing
  ├── Enum values valid?              → 400 if wrong
  ├── Type matches (string/int)?      → 400 if wrong type
  ├── min/max constraints satisfied?  → 400 if out of range
  └── Defaults applied?              → spec.size: "small" added if omitted
```

-----

## Next Steps

➡️ [07 — Composite Resources: Namespaced XRs in Depth](../07-composite-resources/README.md)
