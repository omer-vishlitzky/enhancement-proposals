---
title: resource-claims
authors:
  - ovishlit@redhat.com
creation-date: 2026-10-07
last-updated: 2026-10-11
tracking-link:
  - https://redhat.atlassian.net/browse/OSAC-6096
  - https://redhat.atlassian.net/browse/OSAC-6299
see-also:
  - "/enhancements/OSAC-5753-event-log-driven-metering"
  - "/enhancements/OSAC-4500-quota-foundation"
  - "/enhancements/OSAC-2135-caas-bare-metal-worker-provisioning"
  - "/enhancements/OSAC-1433-unified-networking"
  - "/enhancements/OSAC-3702-lvms-vmaas"
  - "/enhancements/OSAC-6037-storage-api-volume-lifecycle"
  - "/enhancements/OSAC-984-create-update-delete-standalone-volumes"
  - "/enhancements/OSAC-4884-volume-api-storage-attach-detach"
replaces: N/A
superseded-by: N/A
---

# Resource claims

## Summary

OSAC has no standard way to say "this resource is used by that one". Each feature records it differently, or not at
all, so metering, quota and usage reports can't rely on it.

This proposal adds one private field, `claim`, to every resource. It names the one resource that holds this one. Only
the system writes it, and the database enforces its rules. Following claims upward ends at a root, a resource nobody
holds. Everything under a root is attributed to the root's tenant and project.

## Motivation

### Terms

| Term | Meaning |
|---|---|
| Claim | B is held by A, its holder, from the moment B is assigned to A until B is released or physically deleted |
| Root | The first resource up a chain of claims that nobody holds. An unheld resource is its own root |
| Request | What a user asks for, such as an IP attachment or a cluster's node sets. Requests aren't held |
| Shared reference | Used by many, held by none, such as a subnet or a security group |
| Active use | What is busy right now, such as the node a volume is mounted on. Not a claim |
| System tenant | OSAC's own tenant, for resources it runs on tenants' behalf, such as worker hosts |
| Shared tenant | The tenant of platform resources that every tenant can reference |

### How claims are recorded today

- **IP attachments:** a dedicated field, `ExternalIP.status.attribution`.
- **NAT gateways:** a field on the gateway, `NATGateway.spec.external_ip`.
- **Automatic IPs:** the `auto-created-for` label and the `owner-reference` annotation.
- **Cluster workers:** the `cluster-order` label and the `owner-reference` annotation, both naming a hub object, and a
  list on the hub, `ClusterOrder.status.workers`.
- **Volumes:** not recorded.

### What this breaks

- **Metering** can't link a worker or a volume to its cluster or VM, so there is no per-VM, per-cluster or
  per-node-set usage (CAP-11, CAP-12). A system-tenant worker can't be attributed to the tenant it serves.
- **Per-project usage is wrong.** Workers, automatic IPs and tenant-cluster volumes land in a default project, not in
  their cluster's or VM's project (CAP-8, CAP-9).
- **Quota** can't count a cluster's workers and volumes against the cluster's tenant without reading the hub.
- **Billing** can't tell that a VM's disk, billed as a volume, is already counted in the VM's own meter.
- **Every new relation** invents its own way to record it, and its own checks.

### Goals

- One way to record that a resource is held by another, for every type, existing and future.
- The database enforces it, and a new type can't ship without it.
- Metering reads claims from events and quota reads them from the database, all by fulfillment ID.
- Usage is attributed to the root's tenant and project, including system-tenant resources held by a tenant's
  resource.

### Non-Goals

- Deciding what is deleted with its holder. That stays each holder's decision.
- Shared use. Claims are only for exclusive holding. Tracking or splitting the use of a subnet, a storage tier, a NAT
  gateway or a multi-attach volume needs its own design.
- Active use and requests. They stay as they are.
- Usage that isn't an OSAC resource, such as KubeVirt workers (OSAC-1589).

## Proposal

### The claim

Every resource's `Metadata` gets a private field, `claim`, naming its holder:

```proto
message ResourceClaim {
  string type = 1;                      // The holder's type, such as "osac.private.v1.Cluster".
  string id = 2;                        // The holder's ID. Type and ID are empty when nothing holds it.
  string part = 3;                      // Optional: which part of the holder uses this resource.
  google.protobuf.Timestamp since = 4;  // When the claim, or the release, started. Set by the database.
}
```

The claim lives on the held resource, never on the holder. A resource has at most one holder, so the claim is one
field, never a list. It is part of every event of the held resource.

Claims replace `ExternalIP.status.attribution`, the worker label and annotation, and `owner-reference` on fulfillment
resources. `auto-created-for` stays, because it says what is deleted with whom, not who uses what.

### Rules

| Rule | Enforced by |
|---|---|
| A claim is set only where there is none and can't be edited; to change it, release it and claim again | Database |
| The holder exists and isn't being deleted when the claim is set | Database |
| A holder can't finish deleting while anything still names it | Database |
| The held resource is in its holder's tenant or in the system tenant, never in the shared tenant | Database |
| Only system components see and write claims, through the private API, by an explicit claim or release | API |
| A claim caused by a request, such as an IP attachment, is written in the same write as the request | Writers |
| When a holder is deleted, what is deleted with it stays held until gone, and what outlives it is released | Writers |

The database also stamps `since`.

If a holder's deletion doesn't delete or release something it holds, the holder stays deleting until it does. A
missed cleanup shows up as a stuck deletion, never as unattributed usage.

### Attribution

At each moment, a resource is attributed to its root, and through the root to the root's tenant and project.
Attribution isn't ownership: a worker in the system tenant is owned by the system tenant and attributed to the cluster
it serves. Readers ignore resources whose root is in the system tenant.

Example: tenant `acme` runs cluster `c-1` in project `web`. Its workers are in the system tenant.

| Time | What happens | Claims |
|---|---|---|
| 10:00 | `acme` creates `c-1` with automatic IPs | `ip-api` and `ip-ingress` held by `c-1` |
| 10:01 | Workers `w-1` and `w-2` are created | Held by `c-1` |
| 10:30 | A PVC in `c-1` creates volume `vol-9` | Held by `c-1` |
| 11:00 | A scale-down starts deleting `w-2` | Still held |
| 11:04 | `w-2` is gone | The claim ends |

`w-2` is attributed to `c-1`, and so to `acme` and `web`, from 10:01 to 11:04. Moving an IP from `vm-7` to `vm-9` is
a release, then a claim; in between, the IP is its own root.

### Relations

| Held | Holder | Written by | When the holder is deleted |
|---|---|---|---|
| BareMetalInstance (worker) | Cluster | The worker controller | Deleted, held until gone |
| ExternalIP (automatic) | ComputeInstance, BareMetalInstance or Cluster | Fulfillment | Deleted, held until gone |
| ExternalIP (attached) | The attachment's target | Fulfillment, with the attachment | Refused until detached |
| ExternalIP (NAT) | NATGateway | Fulfillment, with the gateway | Released, unless deleted with it |
| Volume (CaaS PVC) | Cluster | The cluster's CSI driver | Open question |
| Volume (VM disk) | ComputeInstance | Open question | Deleted, held until gone (OSAC-6037) |

Deleting a holder that still has a manually attached IP is refused until the user detaches it, as the networking
design specifies (OSAC-1433). The detach releases the IP as soon as it starts, so the IP can be attached elsewhere
right away.

`part` names the node set of a worker, the cluster endpoint an IP serves, or the VM disk a volume backs. Workers get
theirs once they have node sets.

### Enforcement

The database checks every write, so no code path can skip the rules.
Besides the rules above, it rejects:

- **A missing claim.** Every resource always states its holder, or that it has none. The public API states "none" for
  tenants' creates, and system creates must state it themselves. A forgotten claim fails instead of silently making
  a root.
- **An unlisted relation.** A claim is accepted only if its pair of types is in the Relations table. Anything new
  needs an explicit new row.

A resource type without these checks fails CI.

### Readers

- **Metering** attributes each resource's usage to its root's tenant and project, and labels it with the holder, the
  part and the root, so usage can be grouped by VM, cluster or node set. It reads the system tenant's events too, to
  follow system resources to their roots.
- **Quota** counts each resource against its root's tenant. At order time, before workers exist, it checks the
  request, such as node set sizes.
- **Billing** sees from the claim that a volume backs a VM's disk, so pricing can avoid charging it twice.

## Alternatives (Not Implemented)

| Alternative | Why not |
|---|---|
| Record it on the holder | A list on every holder, and a uniqueness check for every holder type |
| Record it on both sides | Two copies drift, as the worker label and the hub list already do |
| A label or an annotation | Anyone who can update the resource can change it |
| A separate claim resource | Two writes instead of one, and the held resource's events don't carry the claim |

## Open Questions

1. When a CaaS cluster is deleted, are its PVC volumes deleted with it, or released to the tenant, who can see
   volumes once OSAC-984 ships?
2. Can the hub's CSI driver name a disk's VM when it creates the volume? If not, it creates the volume with no holder,
   and the operator claims it afterwards.
3. Does attaching a standalone volume to a VM or host (OSAC-4884) claim it, as attaching an IP does? A multi-attach
   volume is shared use either way.
4. Should `part` values be checked for each holder type?

---

## Provenance

Committed: commit @ design 0.11.3 - 9b25062, workspace main @ 63b090a

> Authoring phases not recorded this session (commit-time snapshot only).

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"commit_only","workflow":"design","workflow_version":"0.11.3","ai_workflows":"9b25062","source_repo":"63b090a","source_repo_branch":"main","commits_behind_main":0,"commits_ahead_main":0,"main_ref":"main","phases":["commit","commit"],"authoring_modes":["skill"],"context_changed":false,"origin_untracked":false} -->
