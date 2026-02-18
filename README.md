# 🔧 Learning Crossplane Inner Workings (v2)

> A comprehensive learning resource for understanding how **Crossplane v2** works under the hood — covering namespaced XRDs, YAML configuration, Go internals, Terraform integration, and Kubernetes CRDs. Fully updated for the v2 API (`apiextensions.crossplane.io/v2`) with namespaced composite resources and no Claims.

-----

## 📖 About This Repository

This repository is a structured learning journey into **Crossplane v2’s inner workings**. Crossplane v2 extends Kubernetes to enable infrastructure and application provisioning using the Kubernetes API — turning your cluster into a universal control plane, now with a **namespace-first design**.

### What Changed in v2 (Summary)

|Topic                    |v1 Behaviour                           |v2 Behaviour                                 |
|-------------------------|---------------------------------------|---------------------------------------------|
|XRD scope default        |`LegacyCluster` (cluster-scoped)       |`Namespaced`                                 |
|Developer workflow       |Apply a **Claim** (XRC)                |Apply the **XR directly** in a namespace     |
|Claim object             |Required for namespace scoping         |**Removed** — XRs are now namespaced natively|
|Managed Resources        |Cluster-scoped                         |**Namespaced** (cluster-scoped is legacy)    |
|Composition engine       |Native patch-and-transform OR Functions|**Functions only** (native P&T removed)      |
|Connection secrets       |Built into XR/Claim                    |Must be composed explicitly as a `Secret`    |
|Controller config        |`ControllerConfig`                     |`DeploymentRuntimeConfig`                    |
|Crossplane metadata on XR|Mixed into `spec.*`                    |Isolated under `spec.crossplane.*`           |

**Who is this for?**

- Platform engineers building Internal Developer Platforms (IDPs)
- Cloud engineers learning Crossplane v2 for Azure, AWS, or GCP
- DevOps practitioners migrating from Crossplane v1 or from Terraform

-----

## 📂 Directory Structure

```
learning-crossplane-inner-workings/
│
├── README.md                          ← You are here
│
├── docs/
│   ├── 01-overview/
│   │   └── README.md                  ← What is Crossplane v2? What changed?
│   │
│   ├── 02-architecture/
│   │   └── README.md                  ← v2 control loop & component architecture
│   │
│   ├── 03-yaml-xrd/
│   │   ├── README.md                  ← v2 YAML deep dive (XRD, Composition, XR)
│   │   ├── xrd-example.yaml           ← v2 XRD (apiVersion: v2, scope: Namespaced)
│   │   ├── composition-example.yaml   ← v2 Composition (Pipeline functions only)
│   │   └── xr-example.yaml            ← XR applied directly (no Claim)
│   │
│   ├── 04-go-internals/
│   │   └── README.md                  ← Go internals: ExternalClient, reconciler
│   │
│   ├── 05-terraform-integration/
│   │   └── README.md                  ← Crossplane vs Terraform & Upjet
│   │
│   ├── 06-kubernetes-crds/
│   │   ├── README.md                  ← CRDs — the Kubernetes extension mechanism
│   │   └── crd-example.yaml           ← Namespaced managed resource CRD example
│   │
│   ├── 07-composite-resources/
│   │   ├── README.md                  ← Namespaced XRs in depth (v2 model)
│   │   └── composite-azure.yaml       ← Full Azure example (XRD + Composition + XR)
│   │
│   ├── 08-providers/
│   │   ├── README.md                  ← Provider architecture & namespaced MRs
│   │   └── provider-azure.yaml        ← Azure provider configuration
│   │
│   └── 09-hands-on/
│       ├── README.md                  ← Lab overview
│       ├── lab-01-install.md          ← Lab 1: Install Crossplane v2
│       ├── lab-02-provider.md         ← Lab 2: Configure the Azure Provider
│       └── lab-03-composition.md      ← Lab 3: Build a v2 Composition (no Claims)
│
├── diagrams/
│   ├── crossplane-architecture.mmd    ← Mermaid: v2 overall architecture
│   ├── control-loop.mmd               ← Mermaid: reconciliation loop
│   ├── xrd-to-resource.mmd            ← Mermaid: XRD → Managed Resource flow (v2)
│   ├── v1-vs-v2.mmd                   ← Mermaid: v1 Claim model vs v2 namespaced model
│   └── provider-stack.mmd             ← Mermaid: provider internals
│
└── .github/
    └── LEARNING_PATH.md               ← Suggested learning order
```

-----

## 🗺️ Learning Path

```
[1] Overview           → Understand the v2 philosophy and what changed
[2] Architecture       → Learn the v2 namespace-first control loop
[3] YAML & XRDs        → Master v2 XRDs (scope: Namespaced, no claimNames)
[4] Kubernetes CRDs    → Understand the extension mechanism
[5] Composite Resources → Build namespaced platform APIs without Claims
[6] Go Internals       → How providers implement namespaced MRs
[7] Terraform Compare  → Crossplane vs Terraform in 2025
[8] Providers          → Provider architecture & namespaced MR support
[9] Hands-On Labs      → Apply everything in a real cluster
```

-----

## 🧩 v2 Concepts at a Glance

|Term                               |What It Is                             |v2 Scope                                  |
|-----------------------------------|---------------------------------------|------------------------------------------|
|`CompositeResourceDefinition` (XRD)|Defines your platform API schema       |Cluster object (generates namespaced CRD) |
|`CompositeResource` (XR)           |The resource developers create directly|**Namespace-scoped**                      |
|`Composition`                      |Maps XR fields to composed resources   |Cluster object                            |
|`ManagedResource` (MR)             |A single cloud resource                |**Namespace-scoped**                      |
|`Provider`                         |Cloud adapter package                  |Cluster object                            |
|`ProviderConfig`                   |Credentials reference                  |Namespace-scoped or cluster (per provider)|


> **No Claim (XRC) in v2.** The namespace-scoped XR *is* the developer’s request.

-----

## 📊 v2 Architecture: Everything in One Namespace

```
┌─────────────────────────────────────────────────────────────────────┐
│                    team-alpha NAMESPACE                              │
│                                                                     │
│  kubectl apply:              Crossplane creates in same namespace:  │
│  ┌─────────────────┐         ┌─────────────────────────────────┐   │
│  │  Database (XR)  │────────▶│  MR: ResourceGroup              │   │
│  │  (v2, namespaced│         │  MR: FlexibleServer             │   │
│  │   scope)        │         │  MR: FirewallRule                │   │
│  └─────────────────┘         │  Secret: db-connection          │   │
│                               └─────────────────────────────────┘   │
│                                                                     │
│  RBAC governs who can create XRs in this namespace.                 │
│  No cluster-admin needed for developers.                            │
│  All resources visible with: kubectl get all -n team-alpha          │
└─────────────────────────────────────────────────────────────────────┘

Crossplane Core (crossplane-system):
  XRD Controller → reads XRD, generates namespaced CRD
  Composite Reconciler → runs Composition Function pipeline
  Package Manager → installs/upgrades providers and functions

Provider Pod (crossplane-system):
  Watches namespaced MRs → calls Azure/AWS/GCP API
  Writes observed state back to MR.Status.AtProvider
```

-----

## 🚀 Quick Start (Crossplane v2)

```bash
# 1. Install Crossplane v2
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system \
  --create-namespace \
  --version 2.1.0

# 2. Install function-patch-and-transform (required for Compositions)
kubectl apply -f - <<EOF
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata:
  name: function-patch-and-transform
spec:
  package: xpkg.crossplane.io/crossplane-contrib/function-patch-and-transform:v0.8.2
EOF

# 3. Install the Azure provider
kubectl apply -f docs/08-providers/provider-azure.yaml

# 4. Apply the v2 XRD
kubectl apply -f docs/03-yaml-xrd/xrd-example.yaml

# 5. Apply the v2 Composition
kubectl apply -f docs/03-yaml-xrd/composition-example.yaml

# 6. Create a namespace for your team
kubectl create namespace team-alpha

# 7. Apply the XR directly — no Claim needed!
kubectl apply -f docs/03-yaml-xrd/xr-example.yaml -n team-alpha

# 8. Watch it provision
kubectl get database my-app-database -n team-alpha -w
```

-----

## 📚 External References

- [Crossplane v2 What’s New](https://docs.crossplane.io/latest/whats-new/)
- [v2 XRD Reference](https://docs.crossplane.io/latest/composition/composite-resource-definitions/)
- [Composition Functions](https://docs.crossplane.io/latest/composition/compositions/)
- [Upgrade to v2 Guide](https://docs.crossplane.io/latest/guides/upgrade-to-crossplane-v2/)
- [Upbound Marketplace](https://marketplace.upbound.io/)
- [Crossplane GitHub](https://github.com/crossplane/crossplane)

-----

## 🏷️ Tags

`crossplane-v2` `namespaced-xr` `no-claims` `kubernetes` `platform-engineering` `idp` `azure` `compositions` `composition-functions`
