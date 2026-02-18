# Lab 03: Create a v2 Composition (No Claims)

## Objective

Create a v2 XRD with `scope: Namespaced`, a Pipeline-mode Composition using Functions, and provision resources by applying the XR directly in a namespace — no Claim needed.

-----

## Step 1: Apply the v2 XRD

```bash
kubectl apply -f ../../docs/03-yaml-xrd/xrd-example.yaml

# Wait for the XRD to become Established
kubectl wait xrd/databases.platform.mycompany.com \
  --for=condition=Established --timeout=60s

# Verify: only ONE CRD generated (no cluster-scoped XR CRD in v2)
kubectl get crds | grep platform.mycompany.com
# databases.platform.mycompany.com   ← only one, and it's namespaced

kubectl get crd databases.platform.mycompany.com -o jsonpath='{.spec.scope}'
# → Namespaced   ✅

# In v1 you would also see: xdatabases.platform.mycompany.com (Cluster)
# That no longer exists in v2.
```

-----

## Step 2: Apply the v2 Composition

```bash
kubectl apply -f ../../docs/03-yaml-xrd/composition-example.yaml

kubectl get compositions
# NAME                          AGE
# databases-azure-postgresql    5s
```

-----

## Step 3: Apply the XR Directly (No Claim Needed)

```bash
# Create the developer's namespace
kubectl create namespace team-alpha

# Apply the XR directly into the namespace
# (In v1 you would apply a Claim here — in v2 this IS the developer's object)
kubectl apply -f ../../docs/03-yaml-xrd/xr-example.yaml -n team-alpha

# Confirm the XR is namespace-scoped
kubectl get database my-app-database -n team-alpha
# NAME               SYNCED  READY  COMPOSITION                    AGE
# my-app-database    True    False  databases-azure-postgresql     10s
```

-----

## Step 4: Observe v2-Style Status

```bash
# Watch the XR (in the developer's namespace — no separate cluster-scoped object)
kubectl get database my-app-database -n team-alpha -w

# See spec.crossplane.resourceRefs populated by Crossplane
kubectl get database my-app-database -n team-alpha -o yaml | grep -A 20 crossplane:
# spec:
#   crossplane:
#     compositionRef:
#       name: databases-azure-postgresql
#     resourceRefs:
#       - apiVersion: network.azure.upbound.io/v1beta1
#         kind: ResourceGroup
#         name: my-app-database-rg
#         namespace: team-alpha          ← all in the same namespace!
#       - apiVersion: dbforpostgresql...
#         kind: FlexibleServer
#         name: my-app-database-pg
#         namespace: team-alpha

# See all composed resources in the SAME namespace as the XR
kubectl get managed -n team-alpha
# All MRs are right here — no cross-namespace hunting

# Events are also in the same namespace
kubectl get events -n team-alpha --sort-by='.lastTimestamp'
```

-----

## Step 5: Compare to v1 Behaviour (Educational)

```bash
# In v1 you would search for:
kubectl get xdatabases             # cluster-scoped (hidden from developers)
kubectl get databases -n team-alpha # namespace-scoped Claim

# In v2:
kubectl get databases -n team-alpha # THE developer's object — it IS the XR
# (no cluster-scoped twin exists)

# This also means RBAC is simpler:
# v1 required: Role (for Claims) + ClusterRole (for XRs)
# v2 requires: Role in the namespace only
```

-----

## Step 6: Clean Up

```bash
# Delete the XR — this cascades to all composed MRs (all in the same namespace)
kubectl delete database my-app-database -n team-alpha

# Watch all namespaced MRs disappear
kubectl get managed -n team-alpha -w

# Delete the namespace
kubectl delete namespace team-alpha
```

-----

## ✅ Lab 03 Complete

You’ve successfully used Crossplane v2 to:

- Define a namespaced platform API with `apiextensions.crossplane.io/v2` XRD
- Implement it with a Pipeline-mode Composition (Functions only)
- Provision infrastructure as a developer with **one `kubectl apply`** — no Claim, no cluster-scoped objects, no dual-object confusion

Key v2 takeaways:

- **One object** for the developer (`Database` XR in their namespace)
- **One namespace** for everything (XR + MRs + Secrets)
- **Standard RBAC** — just a `Role` in the team namespace

🎉 **Congratulations — you understand Crossplane v2!**
