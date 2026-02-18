# Lab 02: Configure the Azure Provider (v2)

## Objective

Install the Azure provider family using `DeploymentRuntimeConfig` (not the removed `ControllerConfig`), configure credentials, and verify namespaced MR support.

-----

## Step 1: Install the Provider Family

```bash
# Apply the provider config (uses DeploymentRuntimeConfig, not ControllerConfig)
kubectl apply -f ../../docs/08-providers/provider-azure.yaml

# Wait for the family provider
kubectl wait provider/provider-family-azure \
  --for=condition=Healthy --timeout=300s

kubectl wait provider/provider-azure-network \
  --for=condition=Healthy --timeout=300s
```

-----

## Step 2: Verify DeploymentRuntimeConfig (v2 Replacement for ControllerConfig)

```bash
# Check DeploymentRuntimeConfig was created (ControllerConfig no longer exists)
kubectl get deploymentruntimeconfigs

# Confirm ControllerConfig is gone in v2
kubectl api-resources | grep controllerconfig
# (no output — ControllerConfig is removed in v2)

# Confirm DeploymentRuntimeConfig is available
kubectl api-resources | grep deploymentruntimeconfig
# deploymentruntimeconfigs  pkg.crossplane.io/v1beta1  false
```

-----

## Step 3: Create Azure Credentials

```bash
export SUBSCRIPTION_ID=$(az account show --query id -o tsv)

az ad sp create-for-rbac \
  --name "crossplane-lab-sp" \
  --role Contributor \
  --scopes "/subscriptions/$SUBSCRIPTION_ID" \
  --sdk-auth > /tmp/azure-creds.json

kubectl create secret generic azure-creds \
  --from-file=credentials=/tmp/azure-creds.json \
  -n crossplane-system
```

-----

## Step 4: Apply ProviderConfig

```bash
kubectl apply -f - <<EOF
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
EOF
```

-----

## Step 5: Test with a Namespaced MR

In v2, MRs are applied in a team namespace, not at cluster scope:

```bash
# Create a team namespace
kubectl create namespace lab-test

# Create a namespaced ResourceGroup MR (v2 style)
kubectl apply -f - -n lab-test <<EOF
apiVersion: network.azure.upbound.io/v1beta1
kind: ResourceGroup
metadata:
  name: crossplane-lab-rg
  namespace: lab-test              # ← namespaced in v2!
spec:
  forProvider:
    location: westeurope
    tags:
      CreatedBy: crossplane-lab-v2
EOF

# Watch the namespaced MR
kubectl get resourcegroup -n lab-test -w

# Verify in Azure
az group show --name crossplane-lab-rg

# Clean up (deletion is namespace-scoped too)
kubectl delete resourcegroup crossplane-lab-rg -n lab-test
kubectl delete namespace lab-test
```

-----

## Step 6: Observe Namespace Isolation

```bash
# Team alpha CANNOT see lab-test resources (namespace isolation)
kubectl get resourcegroups -n team-alpha   # empty or not found
kubectl get resourcegroups -n lab-test     # shows the resource

# Platform team (cluster-admin) can see all namespaces
kubectl get resourcegroups --all-namespaces
```

-----

## ✅ Lab 02 Complete

Key v2 differences observed:

- `DeploymentRuntimeConfig` used instead of the removed `ControllerConfig`
- MRs are applied with `-n <namespace>` (namespace-scoped)
- `kubectl get` commands require `-n <namespace>` or `--all-namespaces`

Proceed to [Lab 03](./lab-03-composition.md).
