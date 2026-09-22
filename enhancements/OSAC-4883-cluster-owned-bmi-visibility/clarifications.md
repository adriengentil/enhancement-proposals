# Clarification Log — OSAC-4883

## Status

- Rounds completed: 3
- Open gaps: 0
- Exit criteria met: Yes

## Round 1 — Scope

### R1.Q1: Feature scope — BMI-only or generic ownership model?

The problem (controller-managed resources that are visible to tenants but protected from tenant mutation) is a general pattern applicable across OSAC resource types. Should this feature define a generic ownership model, or is it scoped to BareMetalInstances only?

#### Answer

BMI-only, with the possibility to extend this pattern globally in the future.

#### Impact

PRD scope is BareMetalInstances. Design must be extensibility-conscious but deliver only the BMI case. Out-of-scope section must call out "generalisation of this ownership model to other resource types" as explicit future work.

#### Decision (D1)

This feature is scoped to BareMetalInstances only. The ownership and operation-restriction model must be designed with future extensibility in mind, but no other resource type is in scope for this feature.

---

## Round 2 — Operations, ownership marker, visibility, and errors

### R2.Q1: Allowed vs. restricted operations on cluster-owned BMIs

When a BMI is cluster-owned, the description blocks "destructive operations" but does not enumerate them. Which operations are blocked, and which are permitted?

#### Answer

The operation allowlist is not hardcoded — it is configured at the time `spec.ownerRef` is set (via the private API). The caller that sets ownership also specifies which operations tenants are permitted to perform. *(Position later refined by D10: the default allowlist is empty — fully read-only — and every permitted operation, including label and annotation updates, must be explicitly granted by the caller.)*

#### Impact

`spec.ownerRef` must carry both the owner identity (type + ID) and the configured operation allowlist. The PRD must describe the allowlist as a per-ownership-instance configuration, not a system-wide constant. Different owners may grant different permissions. Design must define the allowlist schema (e.g., an enumerated set of permitted operations).

#### Decision (D2) — amended

The operation allowlist is configurable per ownership instance. It is specified by the caller when setting `spec.ownerRef` via the private API. All operations on the BMI — including label and annotation updates — are subject to the allowlist. See D10 for the default (empty allowlist = fully read-only).

---

### R2.Q2: Ownership marker representation

How should "cluster-owned" be represented on the BMI resource?

#### Answer

A `spec.ownerRef` field specifying the type of the owning object and its ID.

#### Impact

PRD should describe `spec.ownerRef` as a structured field on BareMetalInstance carrying the owning resource type and ID. Design will define the exact proto shape and immutability rules.

#### Decision (D3)

Cluster ownership is represented by a `spec.ownerRef` field on BareMetalInstance, carrying the owning resource type and its identifier.

---

### R2.Q3: Who can set or clear the ownership marker

Can only internal services set the ownership marker, or can cloud provider admins also set it?

#### Answer

Anyone with private API access — both cloud provider admins and internal services (e.g., CaaS controller).

#### Impact

Private API must expose a way to set `spec.ownerRef` at BMI creation time. Public API must not allow tenants to set or modify this field. Access control for the private API already gates who can call it.

#### Decision (D4) — amended

`spec.ownerRef` is set at BMI creation time via the private API and cannot be cleared or transferred after creation. Ownership is permanent for the lifetime of the resource. Any caller with private API access (cloud provider admin or internal service) may create an owned BMI.

---

### R2.Q4: Tenant visibility default for cluster-owned BMIs

Should cluster-owned BMIs appear in normal tenant list/get operations, or must tenants opt in?

#### Answer

Cluster-owned BMIs appear in normal tenant list/get operations automatically (always visible).

#### Impact

No API filter change needed for visibility. The existing tenant-scoped list/get behavior is sufficient — once a BMI is assigned to the owning tenant (instead of `system`), it appears naturally.

#### Decision (D5)

Cluster-owned BMIs are always visible to the owning tenant in standard list and get operations. No opt-in filter is required.

---

### R2.Q5: Error response for blocked operations

When a tenant attempts a blocked operation on a cluster-owned BMI, what should the API return?

#### Answer

Be consistent with today's behavior.

#### Impact (resolved by code inspection)

Current pattern: `ErrDenied` (DAO layer) → `grpccodes.PermissionDenied` gRPC status with a human-readable `Reason` string that is safe to return to callers. This is the pattern used throughout the server layer for access-control failures. Blocked operations on cluster-owned BMIs must follow the same pattern — `PermissionDenied` with a message that explains the BMI is managed by a cluster (e.g., "this BareMetalInstance is managed by a cluster and cannot be deleted directly").

#### Decision (D6)

Blocked operations on cluster-owned BMIs return `PermissionDenied` (gRPC) / 403 (REST) with a human-readable reason message, consistent with the existing `ErrDenied` pattern. The message must explain why the operation is blocked.

---

## Round 3 — Ownership lifecycle, UI, and non-functional requirements

### R3.Q1: Ownership lifecycle on cluster deletion

When the owning cluster is deleted or decommissioned, what happens to the BMI and its `spec.ownerRef`?

#### Answer

The owner of the BMI is responsible for managing it end-to-end. No user intervention is required — the CaaS controller deletes the BMI directly while it is still owned.

#### Impact

No "clear ownerRef before delete" step is needed. The BMI is deleted while owned; ownership semantics end with the resource.

#### Decision (D7) — amended

Ownership cannot be cleared — the only way to end ownership is to delete the BMI. The owner controller deletes the BMI directly (no release step exists). Complexity reduction: removing the release/transfer capability eliminates the orphaned-ownership edge cases that a clearable ownerRef would create.

---

### R3.Q2: UI representation of cluster-owned BMIs

How should the UI distinguish cluster-owned BMIs from tenant-created ones?

#### Answer

The UI reflects the operation allowlist — for example, the "delete" button is disabled for cluster-owned BMIs. UI behaviour follows the same allowlist as the API.

#### Impact

UI work is in scope. The UI must surface `spec.ownerRef` presence and disable or hide operations blocked by the allowlist (delete, lifecycle actions). Label/annotation editing is available only when explicitly included in the ownership allowlist.

#### Decision (D8)

The UI is in scope. Restricted operations (delete, lifecycle) are disabled/hidden for cluster-owned BMIs. The ownerRef is surfaced in the BMI detail view. Label/annotation editing is available only when explicitly included in the ownership allowlist.

---

### R3.Q3: Audit/event logging for ownership changes

Does OSAC have an existing mechanism to log or emit events for ownership changes?

#### Answer

If something already exists, reuse it and emit an event when ownerRef is set. No backward compatibility requirements.

#### Impact (resolved by code inspection)

OSAC has a Watch-based event pipeline: fulfillment service emits `privatev1.Event` objects consumed by the metering service, which publishes CloudEvents 1.0 to Kafka. This is the only structured event infrastructure currently in place — no separate audit log exists. Ownership establishment events should piggyback on this pipeline. No backward compatibility constraints apply (existing `system`-tenant BMIs need no migration).

#### Decision (D9) — amended

Ownership establishment (`spec.ownerRef` set at creation) must emit an event via the existing Watch-based event pipeline. No "ownership cleared" event exists — BMI deletion is the ownership-end signal, covered by the existing BMI deletion event. No backward compatibility with existing `system`-tenant BMIs is required.

---

### R4.Q1: Default allowlist when none is specified

When `spec.ownerRef` is set without an explicit allowlist, what is the default?

#### Answer

Fully read-only — even label and annotation updates require the caller to explicitly include them in the allowlist. No operation is permitted by default.

#### Impact

Labels/annotations are not implicitly allowed. Every permitted operation must be opt-in. The PRD must state that an empty or absent allowlist means fully read-only (get, list only).

#### Decision (D10)

The default allowlist is empty (fully read-only). Every permitted mutation — including label and annotation updates — must be explicitly granted by the caller when setting `spec.ownerRef`. Omitting the allowlist means tenants can only read the resource.

---

## Remaining Gaps

None.
