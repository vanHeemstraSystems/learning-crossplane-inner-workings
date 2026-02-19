# Azure Virtual WAN — Crossplane v2 Sequencing Mechanism

This diagram shows the **reconciliation loop** that Crossplane uses to sequence the
provisioning of an Azure Virtual WAN topology.  It illustrates why
`policy.fromFieldPath: Required` in `function-patch-and-transform` is mandatory:
each resource depends on an Azure-assigned resource ID that only becomes available
*after* the parent resource has been fully provisioned by Azure.

```mermaid
---
title: Azure Virtual WAN — Crossplane v2 Reconciliation Loop (Sequencing Mechanism)
---
sequenceDiagram
    autonumber

    actor User
    participant XR as XVirtualWan (XR)<br/>namespace: platform-networking
    participant CP as Crossplane<br/>Composition Engine
    participant FPT as function-patch-and-transform<br/>(pipeline step 1)
    participant FAR as function-auto-ready<br/>(pipeline step 2)
    participant AZ as Azure API<br/>(provider-azure-network)

    %% ── Initial apply ────────────────────────────────────────────────────
    User->>XR: kubectl apply -f xr.yaml

    rect rgb(230, 240, 255)
        Note over CP,FAR: RECONCILIATION PASS 1
        XR->>CP: XR observed / enqueued
        CP->>FPT: run pipeline (XR spec + empty status)

        FPT->>AZ: create ResourceGroup (rg-my-vwan-dev)
        Note right of FPT: VirtualWAN submitted<br/>(resourceGroupName known)
        FPT->>AZ: create VirtualWAN (Standard)

        Note over FPT: virtualWanId absent in XR status<br/>policy: Required → BLOCK VirtualHub
        Note over FPT: virtualHubId absent in XR status<br/>policy: Required → BLOCK VPNGateway

        FPT->>FAR: pass state (2 MRs pending, 2 blocked)
        FAR->>XR: XR status = Synced=True / Ready=False
    end

    AZ-->>CP: ResourceGroup Ready event
    AZ-->>CP: VirtualWAN Ready event<br/>atProvider.id = /subscriptions/.../virtualWans/my-vwan

    rect rgb(230, 255, 235)
        Note over CP,FAR: RECONCILIATION PASS 2  (triggered by VirtualWAN Ready)
        XR->>CP: XR re-enqueued
        CP->>FPT: run pipeline (XR spec + partial status)

        Note over FPT: ToCompositeFieldPath patches<br/>VirtualWAN.status.atProvider.id<br/>→ XR.status.virtualWanId ✓

        FPT->>AZ: create VirtualHub (10.100.0.0/23)<br/>virtualWanId = resolved ✓

        Note over FPT: virtualHubId still absent<br/>policy: Required → BLOCK VPNGateway

        FPT->>FAR: pass state (3 MRs pending, 1 blocked)
        FAR->>XR: XR status = Synced=True / Ready=False
    end

    AZ-->>CP: VirtualHub Ready event<br/>atProvider.id = /subscriptions/.../virtualHubs/my-vwan-hub

    rect rgb(255, 248, 220)
        Note over CP,FAR: RECONCILIATION PASS 3  (triggered by VirtualHub Ready)
        XR->>CP: XR re-enqueued
        CP->>FPT: run pipeline (XR spec + richer status)

        Note over FPT: ToCompositeFieldPath patches<br/>VirtualHub.status.atProvider.id<br/>→ XR.status.virtualHubId ✓

        FPT->>AZ: create VPNGateway (scaleUnit: 1)<br/>virtualHubId = resolved ✓

        FPT->>FAR: pass state (4 MRs pending, 0 blocked)
        FAR->>XR: XR status = Synced=True / Ready=False
    end

    AZ-->>CP: VPNGateway Ready event

    rect rgb(240, 255, 240)
        Note over CP,FAR: RECONCILIATION PASS 4  (triggered by VPNGateway Ready)
        XR->>CP: XR re-enqueued
        CP->>FPT: run pipeline (all MRs ready)
        FPT->>FAR: pass state (4 MRs ready, 0 blocked)
        FAR->>XR: XR status = Synced=True / Ready=True ✓
        XR-->>User: kubectl get xvirtualwan → READY=True
    end

    %% ── Cleanup ──────────────────────────────────────────────────────────
    Note over User,AZ: CLEANUP
    User->>XR: kubectl delete xvirtualwan my-vwan
    XR->>CP: deletion observed
    CP->>AZ: delete VPNGateway
    CP->>AZ: delete VirtualHub
    CP->>AZ: delete VirtualWAN
    CP->>AZ: delete ResourceGroup
    AZ-->>XR: all resources deleted
    XR-->>User: XR removed from cluster
```

## Key Observations

|Pass|What happens                                                                         |Gate released                       |
|----|-------------------------------------------------------------------------------------|------------------------------------|
|1   |`ResourceGroup` and `VirtualWAN` submitted; `VirtualHub` and `VPNGateway` **blocked**|—                                   |
|2   |`VirtualWAN` Azure ID written to XR status; `VirtualHub` **unblocked**               |`policy: Required` on `virtualWanId`|
|3   |`VirtualHub` Azure ID written to XR status; `VPNGateway` **unblocked**               |`policy: Required` on `virtualHubId`|
|4   |All 4 MRs ready; `function-auto-ready` marks XR `Ready=True`                         |—                                   |

## The Critical Patches

```yaml
# On VirtualWAN — writes Azure ID up to XR status
- type: ToCompositeFieldPath
  fromFieldPath: status.atProvider.id
  toFieldPath: status.virtualWanId

# On VirtualHub — reads that ID, blocks if absent
- type: FromCompositeFieldPath
  fromFieldPath: status.virtualWanId
  toFieldPath: spec.forProvider.virtualWanId
  policy:
    fromFieldPath: Required   # ← converts a failure into a controlled wait
```

Without `policy.fromFieldPath: Required` Crossplane would submit `VirtualHub` immediately
with an empty `virtualWanId`, the Azure API would reject it, and the composition would
enter a permanent error state instead of patiently sequencing through the dependency chain.

## References

- [Lab 04 — Azure Virtual WAN Composition](../docs/09-handson/lab-04-azure-virtual-wan.md)
- [Azure Virtual WAN Network Topology Diagram](./azure-virtual-wan-topology.svg)
- [function-patch-and-transform — fromFieldPath policy](https://github.com/crossplane-contrib/function-patch-and-transform#policy)
- [Crossplane — Composition Pipeline Mode](https://docs.crossplane.io/latest/concepts/compositions/)
