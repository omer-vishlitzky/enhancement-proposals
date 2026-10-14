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

This design defines two relations. A resource that the system created for one other resource names it in a new
field, `created_for`, and its usage is attributed to that resource's tenant and project. Any other resource is
standalone, and its usage is attributed to its own tenant and project, attached or not. Today `created_for` covers a
CaaS cluster's workers.

## Proposal

### Two relations

Every resource is either created for one other resource, or standalone. This is decided once, at creation.

| | Created for | Standalone |
|---|---|---|
| Meaning | The system created it for one resource, which it serves for its whole life | Any other resource |
| Example | A cluster's worker | A VM, or a user's IP attached to it |
| Recorded by | `created_for`, on the created resource | The attachment, while it is attached |
| Link to the other resource | Fixed for its whole life | Changes with each attach and detach |
| Deletion | With the resource it serves | As its own design says |

Following `created_for` upward always ends at a standalone resource, called the top resource. An attachment is any
record of a resource using a standalone one: an IP attachment, a NAT gateway's IP, or a volume attachment (OSAC-4884).

### Where usage goes

A resource's usage is attributed to one tenant and project. It is also included in the total usage that metering
reports for a resource, such as the full cost of a VM or a cluster (CAP-11, CAP-12).

| Resource | Usage attributed to | Included in the total usage of |
|---|---|---|
| Standalone, unattached | Its own tenant and project | Itself |
| Standalone, attached to X | Its own tenant and project, which are also X's | Itself, and X while attached |
| Created for X | X's tenant and project | X |

Totals add up along `created_for`: anything in a worker's total, such as an IP attached to it, is also in its
cluster's total.

Usage follows `created_for` because `created_for` is fixed for life. An attached resource keeps its own tenant and
project because attachments change. A resource's tenant and project are fixed at creation, so its attribution is
fixed for its whole life.

### What it covers today

Today `created_for` covers one case: the workers that the CaaS worker controller creates for a cluster's node sets.
The distinction holds for every resource, and any future resource that the system creates for one other resource
uses `created_for` too.

A resource is created for another only when it serves that resource for its whole life. Automatic IPs and volumes can
leave or outlive the resource they serve, so they are standalone and use attachments, like a user's IP or volume:

- A tenant can delete an automatic IP's attachment, and a VM whose IP cleanup fails is deleted anyway, leaving the IP
  (OSAC-1433, FR-11).
- OSAC-4884 attaches VM disks, and the CSI driver attaches CaaS volumes the same way. A retained CaaS volume can move
  to a new cluster (OSAC-2872).

Default networking serves a whole tenant, so it lives in the tenant's default project.

### Example

Project `web` of tenant `acme` has cluster `c-1` with worker `w-1`, and VM `vm-7` with the user's IP `ip-9` attached.

| Resource | `created_for` | Usage attributed to | Included in the total usage of |
|---|---|---|---|
| `c-1` | Standalone | `acme`, `web` | `c-1` |
| `w-1` | `c-1` | `acme`, `web` | `c-1` |
| `vm-7` | Standalone | `acme`, `web` | `vm-7` |
| `ip-9` | Standalone | `acme`, `web` | `ip-9`, and `vm-7` while attached |

Moving `ip-9` to `vm-8` moves it from `vm-7`'s total to `vm-8`'s, and its usage stays attributed to `acme`, `web`.
Deleting `c-1` deletes `w-1`, and `c-1` finishes deleting after `w-1` is gone.

## Motivation

Today a worker records its cluster in the `cluster-order` label and the `owner-reference` annotation, and both name
the cluster's hub object. This breaks metering and quota:

- **Cluster totals.** Metering can't link a worker to its cluster, so a cluster's total usage (CAP-12) and its usage
  per node set (OSAC-985) are unavailable. Quota can't tell which workers back a cluster (OSAC-4500).
- **Wrong projects.** Workers, automatic IPs and CaaS volumes are created in the default project, while the resource
  they serve may be in another one, so per-project usage is wrong (CAP-8, CAP-9).
- **A bug: attachments cross projects.** API.md and OSAC-1330 say a local reference resolves in the same tenant and
  project, but the code checks only the IP's tenant. An automatic IP's attachment crosses projects whenever its VM is
  outside the default project.

Existing mechanisms can't record `created_for`:

- **Typed references** (OSAC-1330) say that one resource refers to another, such as a VM to its template. A resource
  has many of them, and none says "this resource exists for that one". A reference reaches only its own tenant and
  the shared one, and deleting a referenced resource is refused.
- **The label and the annotation** are user-controlled metadata, which [API.md][api] forbids for relations the
  system depends on. Making them immutable and system-only would make them a relation the system depends on, which
  API.md requires to be a field. This design is that field.

## Behavior changes

1. **Worker usage** is attributed to its cluster's tenant and project (today: the default project of its tenant).
2. **Worker project.** A worker in its cluster's tenant lives in the cluster's project (today: the default project).
3. **Worker link.** A worker names its cluster in `created_for` (today: a label and an annotation naming the
   cluster's hub object).
4. **Worker deletion.** Only the worker controller deletes a worker (today: a user can delete one directly). The
   database keeps a cluster from finishing its deletion while it has workers.
5. **Cluster totals.** Metering reports a cluster's total usage, with its workers, per node set (CAP-12, OSAC-985).
6. **Automatic IPs** live in their resource's project, and their usage is attributed there (today: the default
   project).
7. **CaaS volumes** live in their cluster's project, through the StorageClass `project` parameter that the CSI driver
   already reads (today: the default project). VM disks live in their VM's project (OSAC-6037).
8. **Attachments** join resources in one tenant and project (today: any project of the tenant).
9. **System creates** state `created_for`, or `standalone`, on every resource that metering or quota counts.
10. **References.** `created_for` on a system-tenant resource can name a resource in another tenant (OSAC-1330).

## Goals

- Metering and quota read `created_for` the same way on every type, now and for future types.
- No assumption about which tenant a created resource lives in. Workers live in their cluster's tenant today, and
  OSAC-2135's design puts them in the system tenant.
- The database and the API enforce the rules, and CI fails a type that skips them.

## Non-Goals

- Splitting a standalone resource's usage among the resources it is attached to.
- Changing how attached resources are deleted.
- Attaching across projects, such as sharing a volume between projects.
- Volumes in a VM's or cluster's total usage before OSAC-4884 adds volume attachments.
- Usage that isn't an OSAC resource, such as KubeVirt workers (OSAC-1589).
- Migrating existing deployments. This is greenfield.

## Design Details

### The `created_for` field

Every type that metering or quota counts gets a field, `created_for`. It names the resource this one was created
for, called its target, as a typed reference, or it says `standalone`. Every resource has one or the other:

```text
w-1:   created_for { cluster { id: "c-1" } purpose: "gpu" }
vm-7:  created_for { standalone: true }
```

A typed reference is a protobuf message per target type, such as `ClusterLocalReference`, so the field states both
the target's type and its ID. Each type allows only the targets it needs: a BareMetalInstance can name only a
cluster. A new target is a new API field, so it is always reviewed. One extension to OSAC-1330: a resource in the
system tenant can name a target in another tenant.

`purpose` is optional, and says what the resource serves within its target. Today it holds a worker's node set,
because metering reports a cluster's usage per node set (OSAC-985).

| Rule | Enforced by |
|---|---|
| Only system components set `created_for`, when they create the resource. Users can read it | API |
| Every create through the private API states `created_for`. The public API states `standalone` | Database |
| `created_for` is set once, at creation | Database |
| The target exists and isn't being deleted when the resource is created | Database |
| The resource lives in its target's tenant and project, or in the system tenant | Database |
| Only system components delete the resource | API |
| A target finishes deleting after everything created for it is gone | Database |
| An attachment joins two resources in the same tenant and project | Database |

A create that forgets `created_for` fails, so no resource becomes standalone by mistake. A target exists before
everything created for it, and `created_for` is fixed, so every chain ends at a standalone resource. CI fails any
type that metering or quota counts if it lacks `created_for` or these checks.

Deleting a resource deletes everything created for it, like Kubernetes' foreground deletion. The component that
created each resource deletes it, in its own order: the worker controller removes a worker's node from the cluster
first. A missed cleanup keeps the target visibly in deletion, with all usage still attributed.

### Metering and quota

Metering reads `created_for` from events, as a field known at creation (OSAC-5753). It follows `created_for` to the
top resource, and attributes usage to that resource's tenant and project. Each usage record also names the target,
the purpose and the top resource, so a cluster's total usage and its usage per node set are simple queries.
Attachments stay dimensions of the meters that have them, such as an IP's "attached to".

When a created resource lives in the system tenant, metering reads that tenant's topic like any other. It finds the
top resource's tenant and project by the top resource's ID in its own database, and sends the usage as part of that
tenant's usage. Facts that arrive late, from either topic, are sent in a later round as a difference. Usage whose top
resource is in the system tenant belongs to OSAC, and metering drops it.

Quota reads `created_for` from the database, and counts each resource against its top resource's tenant. Together
with attachments, it is the "which resources back it" that OSAC-4500 assumes. Before the system creates anything for
an order, quota charges the order itself. OSAC-4500 specifies this for a cluster's node sets, and must do the same for
a VM's automatic IP and disks.

## Alternatives (Not Implemented)

- **One untyped field for both relations.** This was the previous revision: a `claim` in every resource's metadata.
  It duplicated typed references, and covering attachments made every relation releasable, with timing rules.
- **Attribute an attached resource to what it is attached to.** Usage would move whenever an attachment moves.
- **Make automatic IPs and volumes created-for.** Their designs let them leave or outlive the resource they serve.
- **Record it on the target, as a list of workers on the cluster.** Every type needs a list, plus a uniqueness check.
- **Kubernetes-style metadata.owner_references.** created_for is the Kubernetes controller owner reference with
foreground deletion. It drops the list and the edits: several owners leave no single tenant and project to attribute
usage to, and adopting or orphaning would move usage between projects. If several owners are ever needed, the
field can become a list without breaking the wire format.

## Open Questions

1. Should `purpose` be checked against its target, such as `gpu` against the cluster's node sets?
2. If workers move to the system tenant, a CaaS volume attached to a worker would cross tenants. Should OSAC-4884 then
   attach CaaS volumes to the cluster?

## Test Plan

**Fulfillment, integration tests against PostgreSQL:**

- **One relation.** A private create without `created_for` fails. A user's create is standalone.
- **Set once.** An update that changes `created_for` fails, and so does a user's attempt to set it.
- **Live target.** Creating a worker for a missing or deleting cluster fails. When a create races the cluster's
  deletion, either the create fails or the worker is deleted with the cluster.
- **Boundaries.** A worker in a different tenant or project from its cluster fails, unless it lives in the system
  tenant. A BareMetalInstance naming a VM fails. Attaching an IP to a VM in another project fails.
- **Placement.** A VM's automatic IP and a cluster's CaaS volume live in the VM's or cluster's project.
- **Deletion.** A user's delete of a worker fails. A cluster finishes deleting after its workers are gone.

**Metering, integration tests against Kafka and PostgreSQL, with recorded event logs:**

- A worker is attributed to its cluster's tenant and project, and grouped by cluster and node set. A worker in the
  system tenant gets the same result, including when the system topic lags behind.
- An IP moved from `vm-7` to `vm-9` stays attributed to its own project. Its "attached to" follows the move.
- A deleting cluster's workers stay attributed to it until they are gone.
- An IP attached to a worker is in the worker's total and the cluster's. Usage whose top resource is in the system
  tenant isn't sent.

**Quota and CI:** a cluster's workers count against the cluster's tenant, wherever they live. A type that metering or
quota counts fails the build without `created_for` and its database checks.

**End-to-end, on a deployed hub:** a tenant creates a CaaS cluster, scales it down and deletes it. Each worker's
usage is attributed to the cluster's project and node set, and the cluster finishes deleting after its workers.

[api]: https://github.com/osac-project/osac/blob/main/fulfillment-service/docs/API.md

---

## Provenance

Authored: manual-edit [manual] @ design 0.11.3 - 9b25062, workspace main @ 63b090a
Phases: respond, manual-edit, manual-edit

> This document's phase history does not include an initial /draft — structure was not verified against the template from origin.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"design","workflow_version":"0.11.3","ai_workflows":"9b25062","source_repo":"63b090a","source_repo_branch":"main","commits_behind_main":0,"commits_ahead_main":0,"main_ref":"main","phases":["commit","commit","respond","manual-edit","manual-edit"],"authoring_modes":["manual","skill"],"context_changed":false,"origin_untracked":true} -->
