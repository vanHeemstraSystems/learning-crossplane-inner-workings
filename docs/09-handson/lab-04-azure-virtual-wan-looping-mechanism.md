# Q&A — Looping Mechanism When Provisioning an Azure Virtual WAN with Crossplane XRD v2

**Q: I was told to pay attention to a looping mechanism in a function when provisioning an Azure Virtual WAN with Crossplane XRD version 2. What is this function and why is it required for an Azure Virtual WAN?**

-----

Great question — this is an important subtlety specific to the Azure Virtual WAN topology. Here is exactly what is going on.

## The Function: `function-sequence-by-status` (or the dependency-ordering pattern)

The looping mechanism you were told to pay attention to is the **readiness-gating loop** built into **`function-patch-and-transform`** combined with `policy.fromFieldPath: Required` — but more specifically, if you are using a dedicated function for this, it is **`function-sequenced-creation`** or the pattern implemented via **`function-extra-resources`**.

However, the most likely specific function being referenced is:

## `function-patch-and-transform` with `policy.fromFieldPath: Required`

Here is why this creates an **implicit reconciliation loop**, and why it is **essential for Azure Virtual WAN** specifically.

### The Dependency Chain Problem

Azure Virtual WAN has a strict **three-level hard dependency chain**:

```
ResourceGroup
    └── VirtualWAN  (must reference ResourceGroup name)
            └── VirtualHub  (must reference VirtualWAN's Azure resource ID)
                    └── VPNGateway  (must reference VirtualHub's Azure resource ID)
```

None of these can be created in parallel. Each one needs the **Azure-assigned resource ID** of its parent — not just the Kubernetes object name, but the actual `/subscriptions/.../resourceId` string that only exists *after* Azure has provisioned the parent resource.

### Why This Requires a Loop

Crossplane’s reconciler works like all Kubernetes controllers: it **reconciles continuously**. Every time it runs the composition pipeline, it evaluates the state of all composed resources.

The `policy.fromFieldPath: Required` setting on the `virtualWanId` and `virtualHubId` patches tells Crossplane:

> “If the value at this field path does not exist yet, **block rendering this resource entirely** — don’t create it with an empty/wrong value.”

This means on the **first reconciliation pass**:

1. `ResourceGroup` is submitted to Azure — not yet ready.
1. `VirtualWAN` is submitted — its `status.atProvider.id` is empty, so the `ToCompositeFieldPath` patch writes nothing to XR status yet.
1. `VirtualHub` render is **blocked** by `policy.fromFieldPath: Required` because `status.virtualWanId` on the XR is still empty.
1. `VPNGateway` is similarly blocked.

Then Crossplane **loops back** — it keeps re-reconciling the XR at a steady interval (typically 60 seconds, or immediately when a resource’s status changes). Each loop it checks again:

- Has `VirtualWAN` become ready and populated `status.atProvider.id`? If yes → write it to XR `status.virtualWanId` → now `VirtualHub` can be created.
- Has `VirtualHub` become ready and populated `status.atProvider.id`? If yes → write it to XR `status.virtualHubId` → now `VPNGateway` can be created.

This is the **loop**: the composition pipeline re-runs repeatedly, each pass unlocking the next resource in the dependency chain once its parent has a real Azure ID.

### Why Azure Virtual WAN Specifically Requires This

Most Azure resources can be referenced by **name** within a resource group — for example, a VNet can reference its resource group just by name. But Virtual WAN is different:

- **`VirtualHub.spec.forProvider.virtualWanId`** requires the full Azure resource ID string (e.g. `/subscriptions/abc.../resourceGroups/rg.../providers/Microsoft.Network/virtualWans/my-vwan`) — there is no `virtualWanIdRef` selector that auto-resolves by Kubernetes object name in all provider versions.
- **`VPNGateway.spec.forProvider.virtualHubId`** has the same requirement for the hub’s full ID.

These IDs are only known **after Azure has actually created the resource** and the provider has observed and written them into `status.atProvider.id`. You cannot predict or pre-compute them. This makes the readiness-gating loop not just useful but **mandatory** — without it, the VirtualHub would be submitted to Azure with an empty `virtualWanId` and would immediately fail with an API validation error.

### The Loop in Practice

```
Pass 1:  RG submitted ──────────────────────────── Hub BLOCKED  VPNgw BLOCKED
Pass 2:  RG ready, WAN submitted ────────────────── Hub BLOCKED  VPNgw BLOCKED
Pass 3:  WAN ready → XR.status.virtualWanId set ─── Hub submitted  VPNgw BLOCKED
Pass 4:  Hub ready → XR.status.virtualHubId set ──────────────── VPNgw submitted
Pass 5:  VPNgw ready → function-auto-ready marks XR READY ✓
```

This is why the Azure Virtual WAN topology typically takes 10–20 minutes to fully provision — each pass through the loop corresponds to a real Azure resource reaching the `Ready` state, and Azure itself takes several minutes per resource.

### Practical Implication for the Lab

The two critical patches in the Composition that make this loop work are:

```yaml
# On VirtualWAN — writes Azure ID to XR status
- type: ToCompositeFieldPath
  fromFieldPath: status.atProvider.id
  toFieldPath: status.virtualWanId

# On VirtualHub — reads that ID, blocks if absent
- type: FromCompositeFieldPath
  fromFieldPath: status.virtualWanId
  toFieldPath: spec.forProvider.virtualWanId
  policy:
    fromFieldPath: Required   # ← this is the gate that forces the loop
```

Without `policy.fromFieldPath: Required`, Crossplane would attempt to create the `VirtualHub` immediately with an empty string as the `virtualWanId`, Azure’s API would reject it, and the composition would fail rather than patiently waiting. The `Required` policy converts a potential failure into a **controlled wait** — effectively turning Crossplane’s natural reconciliation loop into a sequencing mechanism.
