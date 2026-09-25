# KEP-NNNN: Multicluster Addon Management API

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Install a metrics agent](#install-a-metrics-agent)
  - [API Resources](#api-resources)
- [Design Details](#design-details)
  - [Relationship to the Observation API](#relationship-to-the-observation-api)
  - [<code>AddonClass</code>](#addonclass)
  - [<code>Addon</code>](#addon)
  - [References and Identity](#references-and-identity)
  - [Reconciliation and Readiness](#reconciliation-and-readiness)
  - [Deletion](#deletion)
  - [Security and Scale](#security-and-scale)
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

This KEP defines a declarative API for one add-on installation on one
ClusterProfile. An operator creates an `Addon` with a desired release. An
`AddonClass` selects the manager and may provide shared
implementation-specific parameters. The selected manager installs the add-on
and writes its observations to `Addon.status`. A common projection controller
publishes one `AddonInstallation` from that status using the separate
[Addon Observation API](../NNNN-addon-observation-api/README.md).

## Motivation

ClusterProfile identifies a cluster and the Addon Observation API lets
consumers read installation state. Neither asks a selected manager to
install an add-on.

### Goals

1. Declare one desired add-on installation on one ClusterProfile.
2. Select one authoritative manager through `AddonClass`, with shared
   class parameters and optional per-Addon parameters.
3. Report effective desired state, observed state, and readiness on the
   Addon itself.
4. Project an Addon's status into one current `AddonInstallation` without
   requiring every selected manager to write that report.
5. Define ClusterProfile replacement and remote cleanup behavior.

### Non-Goals

This KEP does not standardize:

- selection of multiple targets or PlacementDecision consumption;
- Helm, OCI, Git, Kustomize, OLM, or another package format;
- the meaning of a revision or parameters for different classes;
- rollout ordering, concurrency, canary groups, verification gates,
  immutable history, or automatic rollback;
- add-on dependency installation, cross-namespace Addon references, or
  product-specific peer topology.

## Proposal

### User Stories

#### Install a metrics agent

An operator creates one Addon for `cluster-a` and another for `cluster-b`
in the same inventory namespace. Changing one Addon's revision affects
only that cluster. The projection controller publishes each status as an
AddonInstallation for observation consumers.

### API Resources

The proposed API group is `multicluster.x-k8s.io/v1alpha1`.

| Kind | Scope | Writer | Purpose |
| --- | --- | --- | --- |
| `AddonClass` | Cluster | Platform administrator; selected manager writes status | Select a manager and shared parameters |
| `Addon` | Namespace | User writes spec; selected manager writes status | Declare and observe one installation |

## Design Details

### Relationship to the Observation API

The projection controller is the sole writer of AddonInstallation
objects sourced from Addons. For each Addon, it publishes at most one
report in the Addon's namespace with:

| Addon source | AddonInstallation field |
| --- | --- |
| `metadata.uid` | `installationID` and installation ID label |
| `spec.release.name` | `addonName` |
| `spec.release.revision` | `desiredRevision` |
| `status.classRef.controllerName` | `managerName` |
| `metadata` | `sourceRef` with group, kind, namespace, name, and UID |
| `status.clusterProfileRef` | `clusterProfileRef` with name and UID |
| `status.desiredStateID` | `desiredStateID` |
| `status.observedRevision`, `observedStateID`, `lastObservedTime`, `Ready`, and manager-specific conditions | Corresponding report status |

The projection controller uses a deterministic name and an owner reference
to the same-namespace Addon. It publishes `Ready=True` only when the
Addon's `Ready` condition is `True` and its status meets the other
requirements in [Reconciliation and Readiness](#reconciliation-and-readiness).
While the Addon status is not current or is incomplete, the report uses
`Ready=Unknown` and does not publish stale manager-specific conditions as
current. When the resolved ClusterProfile UID changes, it deletes the old
report and waits for its deletion before publishing a report for the new
UID.

### `AddonClass`

`AddonClass` follows the Gateway API class/controller pattern.

| Field | Required | Meaning |
| --- | --- | --- |
| `spec.controllerName` | Yes | Immutable domain-prefixed path identifying the selected manager |
| `spec.parametersRef` | No | One typed implementation-specific parameters object |
| `status.conditions` | No | Manager-owned `Accepted` condition |

`parametersRef` contains `group`, `kind`, and `name`. It also contains
`namespace` when the referenced kind is namespaced and omits it for a
cluster-scoped kind. A missing, unsupported, or invalid parameters object
makes `AddonClass` `Accepted=False` with reason `InvalidParameters`.

Class parameters provide creation-time defaults. At the first successful
acceptance of an Addon, the selected manager records the effective class
parameter values in persistent manager-owned state before installing
anything. Later changes to the Class or its parameters object do not
automatically change that Addon.
`Addon` parameters can override class defaults; the selected manager
documents how the two objects combine.

The selected manager should keep a finalizer on a Class while Addons
use it.

### `Addon`

An Addon lives in the same namespace as its target ClusterProfile.

| Field | Required | Meaning |
| --- | --- | --- |
| `spec.className` | Yes | Selects an AddonClass; immutable |
| `spec.clusterProfileRef.name` | Yes | Names the target ClusterProfile; immutable |
| `spec.release.name` | Yes | Domain-prefixed capability identity; immutable |
| `spec.release.revision` | Yes | Opaque requested revision; mutable |
| `spec.parametersRef` | No | Typed implementation-specific object in the Addon's namespace |
| `spec.deletionPolicy` | No | `Delete` (default) or `Orphan` when the Addon is deleted |

`spec.parametersRef` contains `group`, `kind`, and `name`. A change to
that reference or its contents changes the effective desired state of
this Addon.

`Addon.status` contains:

| Field | Meaning |
| --- | --- |
| `observedGeneration` | Latest Addon generation handled by the manager |
| `classRef{name,uid,controllerName}` | Exact Class identity accepted by this Addon |
| `clusterProfileRef{name,uid}` | Exact ClusterProfile identity currently managed |
| `desiredStateID` | Effective desired-state identity |
| `observedRevision`, `observedStateID` | Revision and effective state observed by the manager |
| `lastObservedTime` | Last substantive observation |
| `conditions` | `Accepted`, `ResolvedRefs`, `Ready`, and manager-specific conditions |

Conditions use `metav1.Condition` and carry the Addon's
`metadata.generation` in `observedGeneration`. Manager-specific condition
types are domain-prefixed. `desiredStateID` and `observedStateID` have
the meaning defined in the
[Addon Observation API](../NNNN-addon-observation-api/README.md#identity-and-ownership).

### References and Identity

`ResolvedRefs` indicates whether the Class, ClusterProfile, and Addon
parameters exist and can be used. `Accepted=False` means the selected
Class cannot manage this Addon, including when its recorded Class UID no
longer matches.

A recreated ClusterProfile with the same name but a new UID is a new
target. The manager first cleans up the installation associated with the
old UID. It then updates `status.clusterProfileRef` to the new UID and
reconciles the requested installation. During this transition `Ready` is
`False` or `Unknown`. The Addon object and its UID remain unchanged.

### Reconciliation and Readiness

The Addon `Ready` condition has the meaning defined for the
[AddonInstallation `Ready` condition](../NNNN-addon-observation-api/README.md#addoninstallation).
Addon status is current when `status.observedGeneration` and the
`Ready` condition's `observedGeneration` equal `metadata.generation`.
`Ready=True` also requires current status, `Accepted=True`,
`ResolvedRefs=True`, and `status.observedRevision` equal to
`spec.release.revision`.

### Deletion

The selected manager adds its finalizer before making remote changes.
`deletionPolicy: Delete` requires remote removal before it clears the
finalizer. While cleanup is blocked, the Addon reports `Ready=False`
with reason `CleanupBlocked`. `deletionPolicy: Orphan` stops management
and leaves the remote installation in place. During pending deletion,
an authorized user may change `Delete` to `Orphan`.

Loss of the manager's access to the target cluster does not request
uninstallation.

### Security and Scale

Permission to create an Addon authorizes use of ClusterProfiles and
Addon parameters in its namespace for this Alpha API; platform admission
can be stricter. Class parameters are controlled by platform
administrators. The selected manager gets only the permissions its Class
requires. Status must not reveal denied resource contents.

### API Examples

A Class supplies defaults for every metrics Addon that selects it. The
per-Addon parameters object changes only `cluster-a`:

```yaml
apiVersion: observability.example.io/v1alpha1
kind: MetricsAgentParameters
metadata:
  name: production-defaults
  namespace: platform-system
spec:
  scrapeInterval: 30s
---
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: AddonClass
metadata:
  name: metrics-agent
spec:
  controllerName: metrics.example.io/controller
  parametersRef:
    group: observability.example.io
    kind: MetricsAgentParameters
    namespace: platform-system
    name: production-defaults
---
apiVersion: observability.example.io/v1alpha1
kind: MetricsAgentParameters
metadata:
  name: cluster-a-parameters
  namespace: fleet-prod
spec:
  scrapeInterval: 15s
---
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: Addon
metadata:
  name: metrics-agent-cluster-a
  namespace: fleet-prod
  uid: 8df8d26e-5a18-4a31-a79a-b462588c1b77
  generation: 1
spec:
  className: metrics-agent
  clusterProfileRef:
    name: cluster-a
  release:
    name: observability.example.io/metrics-agent
    revision: 2.1.0
  parametersRef:
    group: observability.example.io
    kind: MetricsAgentParameters
    name: cluster-a-parameters
status:
  observedGeneration: 1
  classRef:
    name: metrics-agent
    uid: 36d104be-70c1-4f90-b63b-a88b322f36ac
    controllerName: metrics.example.io/controller
  clusterProfileRef:
    name: cluster-a
    uid: 4614c713-b134-4e09-9d91-61e51f55b352
  desiredStateID: metrics-a-config-v1
  observedRevision: 2.1.0
  observedStateID: metrics-a-config-v1
  lastObservedTime: "2026-09-25T09:00:00Z"
  conditions:
    - type: Accepted
      status: "True"
      reason: Accepted
      message: "AddonClass accepts this Addon"
      observedGeneration: 1
      lastTransitionTime: "2026-09-25T09:00:00Z"
    - type: ResolvedRefs
      status: "True"
      reason: ResolvedRefs
      message: "Class, ClusterProfile, and parameters are resolved"
      observedGeneration: 1
      lastTransitionTime: "2026-09-25T09:00:00Z"
    - type: Ready
      status: "True"
      reason: AddonReady
      message: "Metrics agent is ready on cluster-a"
      observedGeneration: 1
      lastTransitionTime: "2026-09-25T09:00:00Z"
---
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: Addon
metadata:
  name: metrics-agent-cluster-b
  namespace: fleet-prod
spec:
  className: metrics-agent
  clusterProfileRef:
    name: cluster-b
  release:
    name: observability.example.io/metrics-agent
    revision: 2.1.0
```

The projection controller turns the manager's observed status for
`cluster-a` into the AddonInstallation shown in the
[observation examples](../NNNN-addon-observation-api/README.md#api-examples).

### Test Plan

Schema tests cover exactly one local ClusterProfile reference, immutable
identity, typed parameters references, status field shapes, and deletion
policy. Controller conformance tests cover:

1. One Addon producing one same-namespace report with matching source
   and ClusterProfile UIDs.
2. A spec or per-Addon parameters change making old status and the
   projected report stale until the manager observes the new state,
   with condition `observedGeneration` translated to the report
   generation.
3. A Class or Class parameters change leaving existing Addons' effective
   class defaults unchanged, including after restart.
4. A ClusterProfile replacement requiring old-UID cleanup before
   same-Addon rebinding and replacement of the old report.
5. A manager or projection-controller restart, failed remote cleanup,
   and explicit `Orphan` behavior.
6. A recreated Class UID making an existing Addon `Accepted=False`.

### Graduation Criteria

Alpha requires the AddonClass and Addon resource definitions, one
selected manager, a projection controller, and the identity, update,
readiness, and deletion scenarios above. Beta and GA criteria depend on
experience with independent classes and the Observation API.

### Upgrade / Downgrade Strategy

This is a new, out-of-tree API. A future API version needs conversion
for persisted Addons and their Class and ClusterProfile identity
bindings.

### Version Skew Strategy

The projection controller does not publish `Ready=True` from Addon
status fields it does not understand.

## Production Readiness Review Questionnaire

The API has no core Kubernetes feature gate. The Addon finalizer retains
deletion intent through manager outages. A stopped projection controller
leaves last-known reports. Status does not contain Secret contents, logs,
or unbounded peer lists. Object count grows linearly with installations.

## Drawbacks

Projected reports duplicate some Addon status. The projection
controller needs permission to read Addons and write observations in
their namespaces. A selected manager must retain effective creation-time
defaults across restarts.

## Alternatives

A multi-target Addon would require one declaration to track many
independent installation and deletion lifecycles. Requiring every
manager to publish its own AddonInstallation would duplicate the
status mapping.

## References

- [Addon Observation API](../NNNN-addon-observation-api/README.md)
- [ClusterProfile API](../4322-cluster-inventory/README.md)
- [Placement Decision API](../5313-placement-decision-api/README.md)
- [Gateway API class pattern](https://gateway-api.sigs.k8s.io/reference/api-spec/main/spec/)
- [Kubernetes API conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)
