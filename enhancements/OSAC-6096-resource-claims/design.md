---
title: resource-attribution
authors:
  - ovishlit@redhat.com
creation-date: 2026-10-07
last-updated: 2026-10-14
tracking-link:
  - https://redhat.atlassian.net/browse/OSAC-6096
  - https://redhat.atlassian.net/browse/OSAC-6299
see-also:
  - "/enhancements/OSAC-985-metering-and-usage-tracking"
  - "/enhancements/OSAC-5753-event-log-driven-metering"
  - "/enhancements/OSAC-4500-quota-foundation"
  - "/enhancements/OSAC-1330-type-safe-resource-references"
  - "/enhancements/OSAC-2135-caas-bare-metal-worker-provisioning"
  - "/enhancements/OSAC-1433-unified-networking"
  - "/enhancements/OSAC-2872-storage-control-plane"
  - "/enhancements/OSAC-4884-volume-api-storage-attach-detach"
replaces: N/A
superseded-by: N/A
---

# Resource attribution

## Summary

Metering and quota need to know which resource uses which, so that usage is attributed to the right tenant and
project. Today each feature records this differently, or not at all. Without such attribution, the same usage can be
billed twice: a cluster's worker is billed once as the bare-metal instance it is, and again as a node set of its
cluster.

This design defines two relations. A resource that the system created for one other resource lists that resource, its
owner, in a new metadata field, `owner_references`, as Kubernetes does. Its usage is attributed to the owner's tenant
and project. Any other resource is standalone, and its usage is attributed to its own tenant and project, attached or
not. Today owner references cover a CaaS cluster's workers.

## Proposal

### Two relations

Every resource is either owned by one other resource, or standalone. This is decided once, at creation.

| | Owned | Standalone |
|---|---|---|
| Meaning | The system created it for one resource, its owner, which it serves for its whole life | Any other resource |
| Example | A cluster's worker | A VM, or a user's IP attached to it |
| Recorded by | `metadata.owner_references`, on the owned resource | The attachment, while it is attached |
| Link to the other resource | Fixed for its whole life | Changes with each attach and detach |
| Deletion | With its owner | As its own design says |

Following owner references upward always ends at a standalone resource, called the top resource. An attachment is any
record of a resource using a standalone one: an IP attachment, a NAT gateway's IP, or a volume attachment (OSAC-4884).

### Where usage goes

A resource's usage is attributed, and billed, to one tenant and project. Metering also groups usage into totals, such
as the full cost of a VM or a cluster (CAP-11, CAP-12). A total only groups usage, so usage that appears in two totals
is still billed once.

| Resource | Usage attributed to | Included in the total usage of |
|---|---|---|
| Standalone, unattached | Its own tenant and project | Itself |
| Standalone, attached to X | Its own tenant and project, which are also X's | Itself, and X while attached |
| Owned by X | X's tenant and project | X |

Totals add up along owner references: anything in a worker's total, such as an IP attached to it, is also in its
cluster's total.

Usage follows the owner reference because it is fixed for life. An attached resource keeps its own tenant and
project because attachments change. A resource's tenant and project are fixed at creation, so its attribution is
fixed for its whole life.

### What it covers today

Today owner references cover one case: the workers that the CaaS worker controller creates for a cluster's node
sets. The distinction holds for every resource, and any future resource that the system creates for one other
resource gets an owner reference too.

A resource is owned only when it serves its owner for its whole life. Automatic IPs and volumes can leave or outlive
the resource they serve, so they are standalone and use attachments, like a user's IP or volume:

- A tenant can delete an automatic IP's attachment, and a VM whose IP cleanup fails is deleted anyway, leaving the IP
  (OSAC-1433, FR-11).
- OSAC-4884 attaches VM disks, and the CSI driver attaches CaaS volumes the same way. A retained CaaS volume can move
  to a new cluster (OSAC-2872).

Default networking, including its NAT gateway and the gateway's IP, is standalone: the system creates it for a
tenant, not for one resource. It lives in the tenant's default project.

### Example

Project `web` of tenant `acme` has cluster `c-1` with worker `w-1`, VM `vm-7` with the user's IP `ip-9` attached, and
NAT gateway `nat-1` on virtual network `vn-1`, using IP `ip-4`.

| Resource | Owner | Usage attributed to | Included in the total usage of |
|---|---|---|---|
| `c-1` | None | `acme`, `web` | `c-1` |
| `w-1` | `c-1` | `acme`, `web` | `c-1` |
| `vm-7` | None | `acme`, `web` | `vm-7` |
| `ip-9` | None | `acme`, `web` | `ip-9`, and `vm-7` while attached |
| `nat-1` | None | `acme`, `web` | `nat-1` |
| `ip-4` | None | `acme`, `web` | `ip-4`, and `nat-1` while `nat-1` uses it |

Moving `ip-9` to `vm-8` moves it from `vm-7`'s total to `vm-8`'s, and its usage stays attributed to `acme`, `web`.
Deleting `c-1` deletes `w-1`, and `c-1` finishes deleting after `w-1` is gone. `nat-1`, `vn-1` and `ip-4` are in one
project. Deleting `nat-1` removes `ip-4` from `nat-1`'s total, and `ip-4` keeps its own usage until it is deleted.

## Motivation

Today a worker records its cluster in the `cluster-order` label and the `owner-reference` annotation, and both name
the cluster's hub object. This breaks metering and quota:

- **Cluster totals.** Metering can't link a worker to its cluster, so a cluster's total usage (CAP-12) and its usage
  per node set (OSAC-985) are unavailable. Quota can't tell which workers back a cluster (OSAC-4500).
- **A bug: wrong projects.** Workers, automatic IPs and CaaS volumes are created in the default project, while the
  resource they serve may be in another one, so per-project usage is wrong (CAP-8, CAP-9). No design asks for this.
- **A bug: attachments cross projects.** API.md and OSAC-1330 say a local reference resolves in the same tenant and
  project, but the code checks only the tenant, both for IP attachments and for a NAT gateway's virtual network and
  IP. An automatic IP's attachment crosses projects whenever its VM is outside the default project.

Existing mechanisms can't record ownership:

- **Typed references** (OSAC-1330) say that one resource refers to another, such as a VM to its template. A resource
  has many of them, and none says "this resource exists for that one". A reference reaches only its own tenant and
  the shared one, and deleting a referenced resource is refused.
- **The label and the annotation** are user-controlled metadata, which [API.md][api] forbids for relations the
  system depends on. Turning them into a system-only, immutable metadata field, like Kubernetes `ownerReferences`,
  is this design.

## Behavior changes

1. **Worker usage** is attributed to its cluster's tenant and project (today: the default project of its tenant).
2. **Worker project.** A worker in its cluster's tenant lives in the cluster's project (today: the default project).
3. **Worker link.** A worker lists its cluster in `metadata.owner_references` (today: a label and an annotation
   naming the cluster's hub object).
4. **Worker deletion.** Only the worker controller deletes a worker (today: a user can delete one directly). The
   database keeps a cluster from finishing its deletion while it has workers.
5. **Cluster totals.** Metering reports a cluster's total usage, with its workers, per node set (CAP-12, OSAC-985).
6. **Automatic IPs** live in their resource's project, and their usage is attributed there (today: the default
   project).
7. **CaaS volumes** live in their cluster's project, through the StorageClass `project` parameter that the CSI driver
   already reads (today: the default project). VM disks live in their VM's project (OSAC-6037).
8. **Attachments** join resources in one tenant and project, and so do a NAT gateway, its virtual network and its IP
   (today: any project of the tenant). No supported flow crosses projects: OSAC-1330 already forbids it, and
   automatic IPs move to their VM's project.
9. **Metadata.** Every resource gets `metadata.owner_references`, empty for standalone resources (today: none).
10. **Owners across tenants.** A resource in the system tenant can have an owner in another tenant.

## Goals

- Metering and quota read owner references the same way on every type, now and for future types.
- No assumption about which tenant an owned resource lives in. Workers live in their cluster's tenant today, and
  OSAC-2135's design puts them in the system tenant.
- The database and the API enforce the rules, and CI fails a resource table that skips them.

## Non-Goals

- Splitting a standalone resource's usage among the resources it is attached to.
- Changing how attached resources are deleted.
- Attaching across projects, such as sharing a volume between projects.
- Volumes in a VM's or cluster's total usage before OSAC-4884 adds volume attachments.
- Usage that isn't an OSAC resource, such as KubeVirt workers (OSAC-1589).
- Migrating existing deployments. This is greenfield.

## Design Details

### Owner references

Every resource's metadata gets `owner_references`, modeled on Kubernetes `ownerReferences`. A resource that the
system created for another lists that resource, its owner. A standalone resource lists none:

```text
w-1:   owner_references: [{ type: "Cluster", id: "c-1", purpose: "gpu" }]
vm-7:  owner_references: []
```

Each entry names the owner's type and ID, as in Kubernetes. The owner can be of any type, and fulfillment checks
that it exists. `purpose` is optional, and says what the resource serves within its owner. Today it holds a worker's
node set, because metering reports a cluster's usage per node set (OSAC-985). Node sets are part of the cluster's
spec, not resources. If they become resources, a worker is owned by its node set, and the node set by its cluster.

Attribution needs one payer that never changes, so owner references are stricter than in Kubernetes: a resource has
at most one owner, and only the system sets it, once. If several owners are ever needed, the limit can be lifted.

| Rule | Enforced by |
|---|---|
| Only system components set `owner_references`, when they create the resource. Users can read it | API |
| A resource has at most one owner, and its owner never changes | Database |
| A resource can only be created for an owner that exists and isn't being deleted | Database |
| An owned resource lives in its owner's tenant and project, or in the system tenant | Database |
| Only system components delete an owned resource | API |
| An owner finishes deleting only after everything it owns is gone | Database |
| An attachment joins two resources in the same tenant and project | Database |

The database enforces most rules, because some system components write to it directly, without the API: automatic
IPs are created that way. An owner exists before everything it owns, and owner references never change, so every
chain ends at a standalone resource. A forgotten owner reference makes a resource standalone, so each component that
creates owned resources has a test for it. CI fails any resource table without these database checks.

Deleting an owner deletes everything it owns, like Kubernetes' foreground deletion. The component that created each
resource deletes it, in its own order: the worker controller removes a worker's node from the cluster first. A missed
cleanup keeps the owner visibly in deletion, with all usage still attributed.

### Metering and quota

Metering reads owner references from events, as a field known at creation (OSAC-5753). It follows them to the top
resource, and attributes usage to that resource's tenant and project. Each usage record also names the owner, the
purpose and the top resource, so a cluster's total usage and its usage per node set are simple queries.
Attachments stay dimensions of the meters that have them, such as an IP's "attached to".

When an owned resource lives in the system tenant, metering reads that tenant's topic like any other. It finds the
top resource's tenant and project by the top resource's ID in its own database, and sends the usage as part of that
tenant's usage. Facts that arrive late, from either topic, are sent in a later round as a difference. Usage whose top
resource is in the system tenant belongs to OSAC, and metering drops it.

Quota reads owner references from the database, and counts each resource against its top resource's tenant. Together
with attachments, it is the "which resources back it" that OSAC-4500 assumes. Before the system creates anything for
an order, quota charges the order itself. OSAC-4500 specifies this for a cluster's node sets, and must do the same for
a VM's automatic IP and disks.

## Alternatives (Not Implemented)

- **One field for both relations.** This was the previous revision: a `claim` that also covered attachments.
  Attachments change, so every claim had to be releasable, with timing rules.
- **Attribute an attached resource to what it is attached to.** Usage would move whenever an attachment moves.
- **Make automatic IPs and volumes owned.** Their designs let them leave or outlive the resource they serve.
- **Record it on the owner, as a list of workers on the cluster.** Every type needs a list, plus a uniqueness check.

## Open Questions

1. Should `purpose` be checked against its owner, such as `gpu` against the cluster's node sets?
2. If workers move to the system tenant, a CaaS volume attached to a worker would cross tenants. Should OSAC-4884 then
   attach CaaS volumes to the cluster?

## Test Plan

**Fulfillment, integration tests against PostgreSQL:**

- **Default.** A resource created without an owner reference is standalone. The worker controller sets one.
- **Set once.** Changing an owner reference fails, and so do a second owner and a user's attempt to set one.
- **Live owner.** Creating a worker for a missing or deleting cluster fails. When a create races the cluster's
  deletion, either the create fails or the worker is deleted with the cluster.
- **Boundaries.** A worker in a different tenant or project from its cluster fails, unless it lives in the system
  tenant. Attaching an IP to a VM in another project fails, and so does a NAT gateway whose network or IP is in
  another project.
- **Placement.** A VM's automatic IP and a cluster's CaaS volume live in the VM's or cluster's project.
- **Deletion.** A user's delete of a worker fails. A cluster finishes deleting after its workers are gone.

**Metering, integration tests against Kafka and PostgreSQL, with recorded event logs:**

- A worker is attributed to its cluster's tenant and project, and grouped by cluster and node set. A worker in the
  system tenant gets the same result, including when the system topic lags behind.
- An IP moved from `vm-7` to `vm-9` stays attributed to its own project. Its "attached to" follows the move.
- A deleting cluster's workers stay attributed to it until they are gone.
- A NAT gateway's IP is in the gateway's total until the gateway is deleted, and bills in its own project until the
  IP is deleted.
- An IP attached to a worker is in the worker's total and the cluster's. Usage whose top resource is in the system
  tenant isn't sent.

**Quota and CI:** a cluster's workers count against the cluster's tenant, wherever they live. A resource table without
the database checks fails the build.

**End-to-end, on a deployed hub:** a tenant creates a CaaS cluster, scales it down and deletes it. Each worker's
usage is attributed to the cluster's project and node set, and the cluster finishes deleting after its workers.

[api]: https://github.com/osac-project/osac/blob/main/fulfillment-service/docs/API.md

---

## Provenance

Authored: manual-edit [manual] @ design 0.11.3 - 9b25062, workspace main @ 63b090a
Phases: respond, manual-edit, manual-edit

> This document's phase history does not include an initial /draft — structure was not verified against the template from origin.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"design","workflow_version":"0.11.3","ai_workflows":"9b25062","source_repo":"63b090a","source_repo_branch":"main","commits_behind_main":0,"commits_ahead_main":0,"main_ref":"main","phases":["commit","commit","respond","manual-edit","manual-edit"],"authoring_modes":["manual","skill"],"context_changed":false,"origin_untracked":true} -->
