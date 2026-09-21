---
title: csi-aware-scale-from-zero
authors:
  - "@gnufied"
reviewers:
  - "@elmiko"
approvers:
  - "@joelspeed"
api-approvers:
  - None
creation-date: 2026-09-21
last-updated: 2026-09-21
status: provisional
tracking-link:
  - https://redhat.atlassian.net/browse/STOR-3099
see-also:
  - "/enhancements/machine-api/autoscaling-from-to-zero.md"
  - "/enhancements/machine-api/cluster-autoscaler-operator.md"
replaces: []
superseded-by: []
---

# CSI volume limits in scheduler and Cluster Autoscaler

## Summary

Adopt [KEP-5030](https://github.com/kubernetes/enhancements/tree/master/keps/sig-autoscaling/5030-attach-limit-autoscaler)
as an OpenShift Tech Preview. Cluster Autoscaler will include CSI driver
registration and volume attachment limits when simulating new nodes, and the
scheduler can wait for an opted-in CSI driver to register before placing pods
that use it.

## Motivation

Cluster Autoscaler can underestimate the number of nodes needed for pods with
attachable CSI volumes because its simulated nodes lack the corresponding
`CSINode` data. Separately, the scheduler can place pods on a new node before
the CSI driver publishes its attachment limit. The first problem leaves pods
pending after scale-up; the second can place more volumes on a node than the
driver can attach.

### User Stories

* As a cluster administrator, I want autoscaling to account for CSI attachment
  limits so that it creates enough nodes for pending stateful workloads.
* As a cluster administrator, I want the scheduler to wait for a supported CSI
  driver to register on a new node so that pods are not assigned using an
  unknown attachment limit.

### Goals

* Enable KEP-5030 behind `TechPreviewNoUpgrade`.
* Support CSI-aware simulation for node groups with running nodes and for
  supported groups scaled to zero.
* Opt supported OpenShift-managed CSI drivers into the scheduler behavior only
  when Cluster Autoscaler has matching CSI awareness.

### Non-Goals

* Supporting autoscalers other than Cluster Autoscaler.
* Descheduling pods already assigned to an unsuitable node.
* Deriving attachment limits for arbitrary third-party CSI drivers.
* Changing CSI `NodeGetInfo` or attachment enforcement on real nodes.

## Proposal

`TechPreviewNoUpgrade` enables the upstream `VolumeLimitScaling` feature in the
API server and scheduler and starts Cluster Autoscaler with
`--enable-csi-node-aware-scheduling=true`.

Cluster Autoscaler stores `CSINode` data in its cluster snapshot and carries it
into template nodes used for scale-up simulation. For a node group with running
nodes, the template is derived from a representative node and its `CSINode`.
For a group at zero, the cloud provider must supply equivalent data through
`TemplateNodeInfo`.

The scheduler behavior remains opt-in per driver. When
`CSIDriver.spec.preventPodSchedulingIfMissing` is true, the `NodeVolumeLimits`
plugin rejects a node until that driver appears in its `CSINode`. OpenShift CSI
operators set this field for drivers supported by this Tech Preview only while
the feature is enabled.

All three parts must be enabled together. Enabling the scheduler behavior
without CSI-aware Cluster Autoscaler would cause its simulated nodes to be
rejected for CSI-backed pods.

### Workflow Description

1. An administrator enables `TechPreviewNoUpgrade` and configures Cluster
   Autoscaler and a `MachineAutoscaler`.
2. A pod remains pending because existing nodes have insufficient CSI
   attachment capacity.
3. Cluster Autoscaler simulates new nodes with their expected `CSINode` driver
   and limit information and scales the selected node group by the required
   amount.
4. The scheduler does not assign CSI-backed pods to each new node until the
   opted-in driver registers in that node's `CSINode`.

### API Extensions

KEP-5030 adds the feature-gated upstream field
`CSIDriver.spec.preventPodSchedulingIfMissing`. This enhancement adds no
OpenShift API. CSI driver operators, kube-scheduler, and Cluster Autoscaler must
agree on its behavior before a driver opts in.

### Topology Considerations

#### Hypershift / Hosted Control Planes

Hosted control planes use a separate autoscaling deployment and NodePool
template path. They are excluded from the initial Tech Preview and require
separate validation of feature propagation and template `CSINode` data.

#### Standalone Clusters

Standalone clusters using Machine API or Cluster API are the initial target.

#### Single-node Deployments or MicroShift

SNO and MicroShift do not use this machine autoscaling path. The feature adds no
node daemon or steady-state node resource cost.

#### OpenShift Kubernetes Engine

The feature is available when the supported machine autoscaling and CSI driver
operators are present; it has no dependency on an OCP-only workload service.

### Implementation Details/Notes/Constraints

The following downstream integration is required:

* `openshift/kubernetes` carries the upstream API, feature gate, and scheduler
  behavior.
* `openshift/kubernetes-autoscaler` carries upstream CSI snapshot and template
  support. Its Machine API provider must add a synthetic `CSINode` to
  `TemplateNodeInfo`; the Cluster API provider already parses
  `capacity.cluster-autoscaler.kubernetes.io/csi-driver` values in
  `driver=limit` form.
* `openshift/cluster-autoscaler-operator` passes the autoscaler flag with the
  OpenShift feature gate. For MachineSets that have previously run a node, it
  may preserve the minimum observed positive attachment limit in the capacity
  annotation so scale-from-zero remains possible.
* OpenShift CSI driver operators set
  `preventPodSchedulingIfMissing` for the drivers included in the Tech Preview.

Only Ready nodes positively associated with the target MachineSet are valid
sources for a scale-from-zero annotation. Every observed node must report the
driver, and the stored limit is the minimum reported value. A MachineSet that
has never run a node needs an explicit capacity annotation until a deterministic
provider source exists.

### Risks and Mitigations

An overstated template limit can cause insufficient scale-up. Use the minimum
limit reported by representative nodes and invalidate generated data when the
MachineSet template changes.

If scheduler opt-in is enabled while autoscaler CSI awareness or template data
is missing, CSI-backed pods may no longer trigger scale-up. Feature-gate the
components together and test both nonzero and zero-sized node groups before a
driver opts in.

Stale template data can misrepresent a zero-sized group after an instance type
or CSI configuration change. Associate generated data with the MachineSet
template and require a new observation after incompatible changes.

### Drawbacks

Scale-from-zero duplicates runtime `CSINode` information in a durable node-group
template. A group that has never created a node still needs provider knowledge
or explicit configuration.

## Alternatives (Not Implemented)

**Keep scheduler behavior unchanged.** This leaves the registration race that
can overload a newly created node.

**Use a platform-wide fixed limit.** Limits vary by instance type and driver
configuration and may overstate capacity.

**Infer driver registration from cloud hardware.** Hardware attachment support
does not prove that a CSI node plugin will register.

## Open Questions [optional]

1. Which OpenShift-managed attachable CSI drivers and platforms are included in
   the initial Tech Preview?
2. Which component owns generated Machine API capacity annotations, and how are
   its ownership marker and template fingerprint represented?
3. Is a previously observed node required for Machine API scale-from-zero in
   the initial release, or will each provider supply deterministic limits?

## Test Plan

Unit and integration tests cover propagation of `CSINode` data through Cluster
Autoscaler snapshots and templates, parsing of scale-from-zero capacity data,
and scheduler results when the `CSINode` or driver entry is missing.

OpenShift end-to-end tests enable the feature and verify:

* scale-up from a nonzero node group creates enough nodes for pending volumes;
* scale-up from zero works for every supported platform and driver;
* the scheduler waits for driver registration on a new node; and
* disabling the feature restores the previous behavior.

## Graduation Criteria

### Dev Preview -> Tech Preview

* The API server, scheduler, CSI driver operator, and Cluster Autoscaler feature
  switches are wired through `TechPreviewNoUpgrade`.
* End-to-end coverage passes for at least one supported platform and attachable
  CSI driver, including scale from zero.
* Supported drivers and bootstrap limitations are documented.

### Tech Preview -> GA

* The upstream feature reaches GA and is enabled by default in the Kubernetes
  version consumed by OpenShift.
* Blocking CI covers all supported drivers and platforms, including upgrade and
  rollback.
* Scale-up failure metrics and scheduler events provide sufficient diagnostics.

### Removing a deprecated feature

Not applicable.

## Upgrade / Downgrade Strategy

`TechPreviewNoUpgrade` clusters are not supported for upgrade. During
development, rollback must clear `preventPodSchedulingIfMissing` before
disabling autoscaler CSI awareness; otherwise CSI-backed pods may fail
scale-up simulation. Existing pods and attached volumes are unaffected.

## Version Skew Strategy

The scheduler opt-in is safe only when the running Cluster Autoscaler supports
CSI-aware templates. Payload rollout must therefore enable the autoscaler
behavior before setting the field on a `CSIDriver`, and rollback must reverse
that order. The feature does not require kubelet changes.

## Operational Aspects of API Extensions

Operators can use scheduler events together with Cluster Autoscaler
`failed_scale_ups_total` and `unschedulable_pods_count` metrics. No new API type
or steady-state cloud-provider call is introduced.

## Support Procedures

For a CSI-backed pod that does not scale or schedule, inspect the selected node
group's template `CSINode` data, the real node's `CSINode`, the driver's
`preventPodSchedulingIfMissing` setting, and Cluster Autoscaler logs. A mismatch
between the template and real attachment limit is a product defect for a
supported driver and platform.

## Infrastructure Needed [optional]

Presubmit and periodic jobs need a cloud cluster that can scale a worker group
to zero and enough volumes and quota to exceed one node's attachment limit.
