# Lab 01: Install Crossplane v2

## Objective

Install Crossplane v2 on a local `kind` cluster, verify core components, and install the Composition Function packages required for v2 Pipeline mode.

-----

## Step 1: Create a kind Cluster

```bash
kind create cluster \
  --name crossplane-lab \
  --image kindest/node:v1.28.0

kubectl cluster-info --context kind-crossplane-lab
```

-----

## Step 2: Install Crossplane v2 via Helm

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

# Install Crossplane v2 (note: 2.x version)
helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system \
  --create-namespace \
  --version 2.1.0

# Wait for pods
kubectl wait pod \
  --all \
  --for=condition=Ready \
  --namespace=crossplane-system \
  --timeout=300s
```

-----

## Step 3: Install Required Function Packages

In v2, all Composition logic runs via Functions. Install them before creating Compositions.

```bash
# function-patch-and-transform: the standard patching function (replaces native P&T)
kubectl apply -f - <<EOF
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata:
  name: function-patch-and-transform
spec:
  package: xpkg.crossplane.io/crossplane-contrib/function-patch-and-transform:v0.8.2
EOF

# function-auto-ready: marks XRs ready when all composed resources are ready
kubectl apply -f - <<EOF
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata:
  name: function-auto-ready
spec:
  package: xpkg.crossplane.io/crossplane-contrib/function-auto-ready:v0.4.1
EOF

# Wait for functions to be healthy
kubectl wait function/function-patch-and-transform \
  --for=condition=Healthy --timeout=120s
kubectl wait function/function-auto-ready \
  --for=condition=Healthy --timeout=120s
```

-----

## Step 4: Verify Installation

```bash
# Crossplane core pods
kubectl get pods -n crossplane-system
# Expected:
#   crossplane-xxx            1/1 Running
#   crossplane-rbac-xxx       1/1 Running
#   function-patch-xxx        1/1 Running   ← function pod
#   function-auto-ready-xxx   1/1 Running   ← function pod

# Functions installed
kubectl get functions
# NAME                            INSTALLED  HEALTHY
# function-patch-and-transform    True       True
# function-auto-ready             True       True

# Crossplane CRDs (v2 installs fewer core CRDs — Claims CRD is gone)
kubectl get crds | grep crossplane.io
# compositeresourcedefinitions.apiextensions.crossplane.io
# compositions.apiextensions.crossplane.io
# (no compositereourceclaims CRD in v2!)

# Confirm v2 API is available
kubectl api-resources | grep apiextensions.crossplane.io
# compositeresourcedefinitions  xrd,xrds  apiextensions.crossplane.io/v2  false
```

-----

## Step 5: Confirm v2 Scope Behaviour

```bash
# Apply the example XRD
kubectl apply -f ../../docs/03-yaml-xrd/xrd-example.yaml

# Check the generated CRD scope — should be Namespaced (not Cluster)
kubectl get crd databases.platform.mycompany.com \
  -o jsonpath='{.spec.scope}'
# → Namespaced   ✅

# In v1, you would also see:
# xdatabases.platform.mycompany.com  scope: Cluster
# In v2, there is ONLY ONE namespaced CRD — the XR itself
kubectl get crd | grep platform.mycompany.com
# databases.platform.mycompany.com   ← only one CRD
```

-----

## ✅ Lab 01 Complete

You have installed Crossplane v2 with Function support. Note the key differences from v1:

- Version `2.1.0`
- Function packages installed alongside core
- No `ControllerConfig` CRD (removed in v2)
- XRD-generated CRDs are namespace-scoped

Proceed to [Lab 02](./lab-02-provider.md).
