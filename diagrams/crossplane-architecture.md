```mermaid
graph TB
subgraph k8s[“Kubernetes Cluster”]
subgraph cluster_scope[“Cluster-Scoped Objects”]
XRD[“XRD\n(apiextensions.crossplane.io/v2)\nscope: Namespaced\nNO claimNames”]
Comp[“Composition\n(mode: Pipeline\nFunctions only)”]
Prov[“Provider\n(OCI package)”]
Func[“Function\n(e.g. function-patch-and-transform)”]
DRC[“DeploymentRuntimeConfig\n(replaces ControllerConfig)”]
end
    subgraph ns["team-alpha NAMESPACE — all resources live here"]
        XR["Database (XR)\nNamespace-scoped\nDeveloper applies directly\nNO separate Claim"]
        MR1["ManagedResource\nResourceGroup\nNamespace-scoped"]
        MR2["ManagedResource\nFlexibleServer\nNamespace-scoped"]
        Sec["Secret\ndb-connection\n(composed explicitly)"]
    end

    subgraph cp["Crossplane Core (crossplane-system)"]
        XRDCtrl["XRD Controller\n(generates ONE\nnamespaced CRD)"]
        CompReconciler["Composite Reconciler\n(calls Function pipeline\nvia gRPC)"]
        PkgMgr["Package Manager\n(installs Providers\nand Functions)"]
    end

    subgraph prov_pod["Provider Pod (crossplane-system)"]
        ProvCtrl["MR Controllers\n(watch namespaced MRs\nacross all namespaces)"]
        ProvCfg["ProviderConfig\n(credentials)"]
    end

    subgraph fn_pod["Function Pod (crossplane-system)"]
        FnSrv["gRPC Function Server\n(function-patch-and-transform)"]
    end
end

Dev["👤 Developer\nkubectl apply xr.yaml\n-n team-alpha"]
PlatTeam["👷 Platform Team\nkubectl apply xrd.yaml\nkubectl apply composition.yaml"]
Azure["☁️ Azure Cloud API"]

PlatTeam -->|"apply"| XRD
PlatTeam -->|"apply"| Comp
Dev -->|"apply XR directly\n(no Claim!)"| XR

XRD --> XRDCtrl
XRDCtrl -->|"generates ONE\nnamespaced CRD"| ns

XR --> CompReconciler
CompReconciler -->|"gRPC call"| FnSrv
FnSrv -->|"returns desired\ncomposed resources"| CompReconciler
CompReconciler -->|"creates in\nsame namespace"| MR1
CompReconciler -->|"creates in\nsame namespace"| MR2
CompReconciler -->|"creates in\nsame namespace"| Sec

ProvCfg -->|"credentials"| ProvCtrl
MR1 --> ProvCtrl
MR2 --> ProvCtrl
ProvCtrl -->|"REST API"| Azure

style k8s fill:#dbeafe,stroke:#3b82f6
style cluster_scope fill:#f0fdf4,stroke:#16a34a
style ns fill:#fce7f3,stroke:#db2777
style cp fill:#ede9fe,stroke:#7c3aed
style prov_pod fill:#d1fae5,stroke:#059669
style fn_pod fill:#fef3c7,stroke:#d97706
style Dev fill:#fce7f3,stroke:#db2777
style PlatTeam fill:#fce7f3,stroke:#db2777
style Azure fill:#e0f2fe,stroke:#0284c7
```
