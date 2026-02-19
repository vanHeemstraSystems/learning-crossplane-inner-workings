Azure Virtual WAN Composition with Crossplane v2

Executive Summary

The transition to Crossplane v2 introduces significant architectural shifts in how cloud infrastructure is composed and managed. This briefing examines the implementation of an Azure Virtual WAN (vWAN) topology using Crossplane v2’s "Pipeline" mode. Key advancements in this version include the move toward namespace-scoped Composite Resource Definitions (XRDs), the deprecation of Claims in favor of direct Composite Resource (XR) interaction, and the replacement of the classic resource list with an ordered pipeline of specialized Functions. The following analysis details the configuration of a comprehensive Azure network stack comprising a Resource Group, Virtual WAN, Virtual Hub, and VPN Gateway, orchestrated through a function-driven reconciliation loop.

Architectural Evolution in Crossplane v2

Crossplane v2 departs from "classic" composition methods by introducing a more flexible, function-based architecture. The following table summarizes the critical differences applied in modern compositions:

Feature	Crossplane v2 Specification	Change Description
XRD Scope	spec.scope: Namespaced	XRs reside within a specific namespace rather than cluster-wide.
Claims	Removed / Omitted	The claimNames block is absent; consumers interact with XRs directly.
Composition Mode	mode: Pipeline	Replaces the static resources list with an ordered list of Functions.
Logic Execution	Functions	Logic is offloaded to externalized functions (e.g., patch-and-transform).

Core Components and Objects

The Azure Virtual WAN composition utilizes several distinct Kubernetes objects to define the API and the provisioning logic.

1. Required Functions

Functions are the execution engines of the Crossplane v2 pipeline. Two specific functions are required for this topology:

* function-patch-and-transform (v0.8.0): Enables pipeline-mode patching, allowing data to move between the XR and composed resources.
* function-auto-ready (v0.3.0): Automatically marks the XR as "Ready" only once all underlying composed resources have reached a ready state.

2. CompositeResourceDefinition (XRD)

The XRD defines the schema for the XVirtualWan API. It exposes specific parameters to the consumer while maintaining a namespaced scope. Key parameters include:

* Location: The Azure region (e.g., "West Europe").
* Resource Group Name: The identifier for the Azure container.
* WAN Type: SKU selection (Basic or Standard).
* Hub Address Prefix: CIDR block for the Virtual Hub (e.g., "10.0.0.0/23").
* VPN Gateway Scale Unit: Integer defining the gateway's capacity (Range: 1-10).

3. Composition

The Composition wires together the managed resources using the Pipeline mode. It dictates the order of operations and the flow of data between the following Azure resources:

* Resource Group
* Virtual WAN
* Virtual Hub
* VPN Gateway

Technical Implementation Analysis

The Pipeline Workflow

The reconciliation process follows a strict sequence within the composition.yaml:

1. Step 1: Patch-and-Transform:
  * This step uses function-patch-and-transform to map XR parameters (like location and tags) to the specific fields of the managed resources.
  * Status Writeback: Using ToCompositeFieldPath, the Azure Resource IDs of the Virtual WAN and Virtual Hub are written back to the XR's status fields (status.virtualWanId and status.virtualHubId).
  * Cross-Resource Referencing: The pipeline facilitates dependencies. For instance, the Virtual Hub references the Virtual WAN ID from the XR status, and the VPN Gateway references the Virtual Hub ID.
2. Step 2: Auto-Ready:
  * This step invokes function-auto-ready to monitor the health of the four managed resources, ensuring the top-level XR accurately reflects the collective state of the infrastructure.

Resource Interdependencies

The composition ensures that resources are created with the necessary context from their parents. The table below outlines the mapping logic:

Composed Resource	Dependency	Reference Method
Virtual WAN	Resource Group	Patched via spec.parameters.resourceGroupName
Virtual Hub	Virtual WAN	Referenced via status.virtualWanId
VPN Gateway	Virtual Hub	Referenced via status.virtualHubId

Deployment and Lifecycle Observations

Provisioning Timelines

Deploying a full Virtual WAN topology with a VPN Gateway is a time-intensive process. Based on technical documentation, users should expect a duration of 10 to 20 minutes for the Azure environment to reach a fully provisioned state.

Monitoring and Verification

The state of the deployment can be monitored through standard Kubernetes CLI tools:

* XR Progress: Monitored via kubectl get xvirtualwan, which displays SYNCED and READY status.
* Managed Resource Inspection: Individual Azure resources (resourcegroups, virtualwans, virtualhubs, vpngateways) can be verified using labels (e.g., crossplane.io/composite=[XR_NAME]).
* Status Writeback Verification: Inspecting the XR status reveals the specific Azure resource IDs once they are assigned by the provider.

Cleanup and Deletion

Crossplane v2 maintains the principle of cascade deletion. Deleting the top-level XVirtualWan XR triggers an automatic, ordered deletion of all associated managed resources in Azure. This ensures that no orphaned resources remain in the cloud provider environment after the composite resource is removed.
