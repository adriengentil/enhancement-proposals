# Cluster-Owned BareMetalInstance Visibility and Operation Control

| Field       | Value   |
|-------------|---------|
| Author(s)   | Adrien Gentil |
| Jira        | https://redhat.atlassian.net/browse/OSAC-4883 |
| Date        | 2026-09-22 |

## Problem Statement

BareMetalInstances (BMIs) provisioned for CaaS cluster worker nodes are currently hidden under the system tenant, making them invisible to the tenant that owns the cluster. Tenants cannot see the compute resources backing their clusters, and billing and quota attribution is inaccurate because BMI consumption appears under the system tenant rather than the owning tenant. The established industry pattern — AWS EKS, GCP GKE, Azure AKS — is to make underlying instances visible to tenants while restricting destructive operations, not to hide them entirely. Without this change, metering and quota enforcement cannot correctly attribute bare metal compute consumption to the tenants who own the clusters that consume it.

## In Scope

- BareMetalInstances assigned to a tenant are visible to that tenant in standard list and detail views across the UI, CLI, and API
- A per-resource ownership designation for BareMetalInstances: an authorized caller marks a BMI as owned by a specific resource (e.g., a cluster) and configures which operations tenants are permitted to perform on it
- A configurable, per-ownership-instance operation allowlist — established when ownership is set — that defines which tenant operations are permitted; by default an owned BMI is fully read-only, and each permitted operation (including label and annotation updates) must be explicitly granted
- Ownership is set at BMI creation time via the private API and cannot be cleared or transferred after creation; tenants cannot set ownership
- An owner resource must belong to the same tenant as the BareMetalInstance it owns — cross-tenant ownership is not permitted
- The UI disables or hides operations that are restricted for owned BMIs, and surfaces ownership information in the BMI detail view
- Ownership establishment at creation produces an observable lifecycle event consumable by downstream systems
- E2E testing covering BMI visibility, allowed operations, and blocked operations for owned BMIs

## Out of Scope

- Extension of the ownership model to other OSAC resource types (ComputeInstance, VirtualNetwork, etc.) — designated as future work
- Migration of existing system-tenant BMIs to owning tenants; no backward compatibility is required
- A dedicated audit log or event store for ownership history — observable lifecycle events are the only event mechanism in scope
- Automated cluster node lifecycle management (provisioning, decommissioning) — delivered by CaaS, not this feature
- Releasing or transferring BMI ownership after creation — ownership is permanent for the lifetime of the resource
- Detection of a deleted or missing owner resource and automated remediation — if the owning resource is deleted, the BMI's ownership state is not automatically updated (ownership cannot be cleared); the owner is responsible for deleting the BMI; an orphaned BMI remains billable and quota-counted, and any authorized private API caller may delete it

## User Stories

### Tenant Admin / Tenant User

- As a Tenant Admin or Tenant User, I want BareMetalInstances backing my CaaS clusters to appear in my standard BareMetalInstance list and detail views, so that I can see the compute resources contributing to my quota and billing without asking an administrator.
- As a Tenant Admin or Tenant User, I want to understand which operations I am permitted to perform on a cluster-owned BareMetalInstance, and receive a clear, actionable message when I attempt a restricted operation, so that I can operate within ownership constraints without encountering opaque failures.
- As a Tenant Admin or Tenant User, I want the UI to disable or hide operations that are restricted for cluster-owned BareMetalInstances (such as delete or power actions), so that I can see at a glance what I can and cannot do before attempting an operation.
- As a Tenant Admin or Tenant User, I want to perform the operations that the cluster owner has explicitly permitted on a cluster-owned BareMetalInstance, so that I retain meaningful access to resources that are visible in my tenant.

### Cloud Provider Admin

- As a Cloud Provider Admin, I want to create a BareMetalInstance with an ownership designation and a configured operation allowlist, so that cluster nodes are visible to tenants while being protected from unintended destructive actions for the full lifetime of the resource.

## Assumptions

- Downstream systems that consume ownership lifecycle events can be extended to handle new ownership event types without requiring separate event infrastructure.

## Dependencies

- **OSAC-2135 (CaaS Bare Metal Worker Node Provisioning):** This feature reverses the OSAC-2135 design decision to hide CaaS-managed BMIs under the system tenant. CaaS provisioning and deprovisioning flows must be updated to assign BMIs to the owning tenant and manage ownership via the private API introduced by this feature. OSAC-4883 design approval must precede the OSAC-2135 implementation changes.
- **OSAC-5086 (Service-resource tenant isolation):** Tenant isolation validation for cross-resource references (e.g., BareMetalInstance network references) is deferred in OSAC-5086 until this feature's ownership and visibility model is approved. OSAC-5086 implementation follows OSAC-4883.
- **OSAC-4500 (Quota Foundation MVP):** Accurate quota attribution for CaaS-backed bare metal compute depends on BMIs being assigned to the owning tenant. Quota enforcement logic must account for the distinction between tenant-created and cluster-owned BMIs.

---

## Provenance

Authored: respond @ prd 0.11.3 - 9b25062, workspace main @ f0a8211
Phases: draft, respond, respond, respond, respond, respond

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"prd","workflow_version":"0.11.3","ai_workflows":"9b25062","source_repo":"f0a8211","source_repo_branch":"main","commits_behind_main":0,"commits_ahead_main":0,"main_ref":"main","phases":["draft","respond","respond","respond","respond","respond"],"authoring_modes":["skill"],"context_changed":false,"origin_untracked":false} -->
