# KEP-NNNN: Multicluster Addon Observation API

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Read installations from different managers](#read-installations-from-different-managers)
    - [Observe a failed peer connection](#observe-a-failed-peer-connection)
  - [API Resources](#api-resources)
- [Design Details](#design-details)
  - [Identity and Ownership](#identity-and-ownership)
  - [<code>AddonInstallation</code>](#addoninstallation)
  - [Current Observations](#current-observations)
  - [Lifecycle and Scale](#lifecycle-and-scale)
  - [API Examples](#api-examples)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [References](#references)
<!-- /toc -->

## Summary

This KEP defines a vendor-neutral, read-only contract for the observed state
of add-ons installed into or for clusters represented by ClusterProfile.
An existing manager can publish `AddonInstallation` without adopting the
related [Addon Management API](../NNNN-addon-management-api/README.md).
For a manager that does use Addon, a common projection controller
projects the manager's `Addon.status` into one AddonInstallation for
that cluster.

A report identifies the desired and observed revision, an opaque identity
for the effective desired and observed state, and a `Ready` condition.

## Motivation

ClusterProfile identifies a cluster, PlacementDecision lists selected
clusters, and Work API can deliver resources. None gives a consumer a
common answer to which add-on is installed on a cluster, what state was
requested, or whether that state is usable. Existing managers expose
different resource kinds and condition meanings.

A successful delivery report alone does not show that a CNI, storage
driver, or service mesh is ready for use.

### Goals

1. Let independent managers publish cluster-level add-on identity, desired
   state, observed state, and readiness in one format.
2. Bind each report to a ClusterProfile UID so a recreated profile cannot
   inherit the previous cluster's result.
3. Include required cross-cluster checks in the affected cluster's readiness.
4. Let consumers reject a result for an earlier desired state or object
   generation while keeping objects bounded in size.

### Non-Goals

This KEP does not standardize:

- user-authored installation intent, package formats, scheduling, delivery,
  or uninstall behavior;
- a rollout strategy, verification runner, dependency controller, or
  readiness gate for another API;
- target set selection, a target set report, or a computed aggregate of
  member readiness;
- product-specific peer topology, health checks, or their implementation;
- a catalog of capabilities, a heartbeat interval, or a universal
  freshness or lease policy.

## Proposal

### User Stories

#### Read installations from different managers

An operator has a metrics agent managed by one system on `cluster-a` and
another on `cluster-b`. A tool shows each cluster's requested revision,
observed revision, and `Ready` condition without understanding either
manager's source resource.

A deployment controller can wait for a CNI installation on an exact
ClusterProfile UID to report the requested state as ready.

#### Observe a failed peer connection

A mesh manager initially reports ready installations on `cluster-a` and
`cluster-b`. Traffic from `cluster-a` to `cluster-b` then fails while the
reverse path works. `cluster-a`'s report changes to `Ready=False` and
`cluster-b`'s may remain `Ready=True`. The reports can be published
directly by the manager or projected from two per-cluster Addons.

### API Resources

The proposed API group is `multicluster.x-k8s.io/v1alpha1`.
`AddonInstallation` is namespaced and has no `spec`. A direct publisher
reports its own intent and observation. For a report sourced from an
Addon, the projection controller defined in the
[Addon Management API](../NNNN-addon-management-api/README.md#relationship-to-the-observation-api)
is the publisher. Creating, updating, or deleting a report does not
request a change to an installation.

## Design Details

### Identity and Ownership

The tuple `(namespace, managerName, installationID)` identifies a logical
installation. `installationID` is a valid Kubernetes label value. A
publisher creates at most one AddonInstallation for a given logical
installation and ClusterProfile UID.

The publisher is the only writer of each object's top-level fields,
status, and deletion. The kind enables the status subresource.

`addonName`, `managerName`, `installationID`, `sourceRef`, and
`clusterProfileRef` are immutable. A recreated source or ClusterProfile
requires a new report.

`desiredStateID` is an opaque equality token for the effective desired
release, parameters, and consumed cluster or shared inputs, scoped to
the manager and installation. It changes whenever an input that can
change installation or a required readiness check changes. Managers
define which topology inputs affect each report. A peer health failure
without an input change does not change either state ID. The publisher
does not reuse one ID for different effective inputs.

### `AddonInstallation`

A report lives in the same namespace as its ClusterProfile, and both
`clusterProfileRef` fields are required. A report has no owner reference
to ClusterProfile.

| Field | Meaning |
| --- | --- |
| `addonName` | Capability identity as a domain-prefixed path |
| `managerName`, `installationID` | Logical manager and installation identity |
| `sourceRef` | Optional source object group, kind, namespace, name, and UID |
| `clusterProfileRef` | ClusterProfile name and UID |
| `desiredRevision` | Requested opaque revision |
| `desiredStateID` | Effective desired-state identity |
| `status.observedRevision` | Revision observed by the manager |
| `status.observedStateID` | Effective state observed by the manager |
| `status.conditions` | Required `Ready` and optional domain-prefixed manager conditions |
| `status.lastObservedTime` | Last substantive observation |
| `status.detailsRef` | Optional same-namespace manager-specific detail |

Conditions use `metav1.Condition` with list-map semantics keyed by type.
A publisher sets each condition's `observedGeneration` to the report's
generation and updates affected conditions together in one status
update. `Ready=True` requires matching desired and observed state IDs
and positive evidence that the advertised capability is usable.
`Ready=False` reports a known failure or an observed state that is not
ready. `Ready=Unknown` reports an inconclusive required check without a
known failure.

For a cross-cluster capability, required checks include paths from the
referenced cluster to its required peers.

### Current Observations

A consumer treats `Ready` as current only when all of the following
hold:

1. The report identifies the expected manager, installation, source UID
   when present, and ClusterProfile UID.
2. The `Ready` condition's `observedGeneration` equals the report's
   `metadata.generation`.
3. `status.observedStateID` equals `desiredStateID`.

A consumer that needs a freshness limit also compares
`lastObservedTime` with its own policy. An absent report means that no
conforming publisher has reported that installation through this API.
Consumers must not infer a complete target set by counting reports.

### Lifecycle and Scale

A publisher creates a report when it starts managing or observing an
installation and deletes it after management ends.

Reports carry the `multicluster.x-k8s.io/installation-id` label. A
report contains no unbounded list of clusters, peer checks, measurements,
or logs. Managers can use `status.detailsRef` for structured peer detail.

### API Examples

An existing metrics manager can publish directly without an Addon:

```yaml
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: AddonInstallation
metadata:
  name: metrics-agent-cluster-b
  namespace: fleet-prod
  generation: 1
  labels:
    multicluster.x-k8s.io/installation-id: direct-metrics
addonName: observability.example.io/metrics-agent
managerName: legacy-metrics.example.io/controller
installationID: direct-metrics
clusterProfileRef:
  name: cluster-b
  uid: 93ab0a21-8b16-46af-a25d-b4ed932c2f13
desiredRevision: 2.1.0
desiredStateID: metrics-b-config-v1
status:
  observedRevision: 2.1.0
  observedStateID: metrics-b-config-v1
  lastObservedTime: "2026-09-25T09:00:00Z"
  conditions:
    - type: Ready
      status: "True"
      reason: AddonReady
      message: "Metrics agent is ready on cluster-b"
      observedGeneration: 1
      lastTransitionTime: "2026-09-25T09:00:00Z"
```

For the `metrics-agent-cluster-a` Addon in the
[management example](../NNNN-addon-management-api/README.md#api-examples),
the projection controller publishes this report:

```yaml
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: AddonInstallation
metadata:
  name: addon-8df8d26e-5a18-4a31-a79a-b462588c1b77
  namespace: fleet-prod
  generation: 1
  labels:
    multicluster.x-k8s.io/installation-id: 8df8d26e-5a18-4a31-a79a-b462588c1b77
  ownerReferences:
    - apiVersion: multicluster.x-k8s.io/v1alpha1
      kind: Addon
      name: metrics-agent-cluster-a
      uid: 8df8d26e-5a18-4a31-a79a-b462588c1b77
addonName: observability.example.io/metrics-agent
managerName: metrics.example.io/controller
installationID: 8df8d26e-5a18-4a31-a79a-b462588c1b77
sourceRef:
  group: multicluster.x-k8s.io
  kind: Addon
  namespace: fleet-prod
  name: metrics-agent-cluster-a
  uid: 8df8d26e-5a18-4a31-a79a-b462588c1b77
clusterProfileRef:
  name: cluster-a
  uid: 4614c713-b134-4e09-9d91-61e51f55b352
desiredRevision: 2.1.0
desiredStateID: metrics-a-config-v1
status:
  observedRevision: 2.1.0
  observedStateID: metrics-a-config-v1
  lastObservedTime: "2026-09-25T09:00:00Z"
  conditions:
    - type: Ready
      status: "True"
      reason: AddonReady
      message: "Metrics agent is ready on cluster-a"
      observedGeneration: 1
      lastTransitionTime: "2026-09-25T09:00:00Z"
```

The mesh Addons `mesh-cluster-a` and `mesh-cluster-b` initially report
`Ready=True`. At 09:05, only traffic from `cluster-a` to `cluster-b`
fails. The projection controller then publishes these reports:

```yaml
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: AddonInstallation
metadata:
  name: addon-2bdf1ce9-2c66-4e1b-9a4c-8a9c0955e650
  namespace: fleet-prod
  generation: 1
  labels:
    multicluster.x-k8s.io/installation-id: 2bdf1ce9-2c66-4e1b-9a4c-8a9c0955e650
addonName: istio.io/multicluster
managerName: mesh.example.io/controller
installationID: 2bdf1ce9-2c66-4e1b-9a4c-8a9c0955e650
sourceRef:
  group: multicluster.x-k8s.io
  kind: Addon
  namespace: fleet-prod
  name: mesh-cluster-a
  uid: 2bdf1ce9-2c66-4e1b-9a4c-8a9c0955e650
clusterProfileRef:
  name: cluster-a
  uid: 4614c713-b134-4e09-9d91-61e51f55b352
desiredRevision: 1.30.0
desiredStateID: mesh-a-topology-v1
status:
  observedRevision: 1.30.0
  observedStateID: mesh-a-topology-v1
  lastObservedTime: "2026-09-25T09:05:00Z"
  conditions:
    - type: Ready
      status: "False"
      reason: PeerConnectivityFailed
      message: "Traffic from cluster-a to cluster-b fails"
      observedGeneration: 1
      lastTransitionTime: "2026-09-25T09:05:00Z"
    - type: mesh.example.io/PeersReachable
      status: "False"
      reason: PeerUnreachable
      message: "cluster-b is unreachable from cluster-a"
      observedGeneration: 1
      lastTransitionTime: "2026-09-25T09:05:00Z"
---
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: AddonInstallation
metadata:
  name: addon-36583a56-c520-4e32-829d-47a138182880
  namespace: fleet-prod
  generation: 1
  labels:
    multicluster.x-k8s.io/installation-id: 36583a56-c520-4e32-829d-47a138182880
addonName: istio.io/multicluster
managerName: mesh.example.io/controller
installationID: 36583a56-c520-4e32-829d-47a138182880
sourceRef:
  group: multicluster.x-k8s.io
  kind: Addon
  namespace: fleet-prod
  name: mesh-cluster-b
  uid: 36583a56-c520-4e32-829d-47a138182880
clusterProfileRef:
  name: cluster-b
  uid: 93ab0a21-8b16-46af-a25d-b4ed932c2f13
desiredRevision: 1.30.0
desiredStateID: mesh-b-topology-v1
status:
  observedRevision: 1.30.0
  observedStateID: mesh-b-topology-v1
  lastObservedTime: "2026-09-25T09:05:00Z"
  conditions:
    - type: Ready
      status: "True"
      reason: AddonReady
      message: "Mesh is usable from cluster-b"
      observedGeneration: 1
      lastTransitionTime: "2026-09-25T09:00:00Z"
    - type: mesh.example.io/PeersReachable
      status: "True"
      reason: AllPeersReachable
      message: "cluster-a is reachable from cluster-b"
      observedGeneration: 1
      lastTransitionTime: "2026-09-25T09:00:00Z"
```

### Test Plan

Schema tests cover identity, a same-namespace ClusterProfile reference
with a UID, condition structure, and immutable source identity.
Publisher and consumer conformance tests cover:

1. A manager publishing directly without Addon or AddonClass.
2. Projected and direct-publisher reports coexisting without identity
   collisions.
3. A revision or manager-specific input change invalidating the
   previous observation.
4. ClusterProfile UID replacement creating a new report without
   reusing the old `Ready` result.
5. A required peer path changing the affected cluster from `Ready=True`
   to `Ready=False` without changing state IDs or generation.
6. A report for a capability with many peers remaining bounded.

### Graduation Criteria

Alpha requires the resource definition, one direct publisher, and a
consumer that reads exact cluster identity and current readiness. Beta
and GA criteria depend on experience with independent publishers and
consumers.

### Upgrade / Downgrade Strategy

This is a new, out-of-tree API. A manager may publish directly alongside
its existing resources without changing installation ownership. Removing
a direct reporting integration removes this common view.

### Version Skew Strategy

An older consumer ignores unknown optional fields and uses only
identity and `Ready` semantics it understands. A future API version
preserves immutable identity and current-observation rules across
conversion.

## Production Readiness Review Questionnaire

The API has no core Kubernetes feature gate. A stopped publisher leaves
last-known reports. Reports contain no credentials. Namespaced RBAC
grants write access only to publishers. Object count grows with
installations.

## Drawbacks

The common `Ready` condition relies on manager evidence. An independent
predicate with no member-cluster meaning cannot be reported separately
through this API.

## Alternatives

Putting add-on state in ClusterProfile status would create a multi-writer
inventory object. Requiring an Addon for every report would prevent
observation-only adoption by existing managers. A separate fleet health
report would duplicate peer checks that are already reflected in
affected clusters' readiness.

## References

- [Addon Management API](../NNNN-addon-management-api/README.md)
- [ClusterProfile API](../4322-cluster-inventory/README.md)
- [Placement Decision API](../5313-placement-decision-api/README.md)
- [Kubernetes API conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)
- [KEP-1623: Standardize Conditions](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/1623-standardize-conditions)
