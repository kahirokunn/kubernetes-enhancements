# KEP-NNNN: Placement Rollout Strategy API

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Users and Resources](#users-and-resources)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [API Types](#api-types)
  - [Resolving Targets](#resolving-targets)
  - [Stages, Partitions, and Update Budgets](#stages-partitions-and-update-budgets)
  - [Steps and Verification](#steps-and-verification)
  - [Validation and Access](#validation-and-access)
  - [API Example: Canary, Staging, Production](#api-example-canary-staging-production)
  - [API Example: Parallel Regions](#api-example-parallel-regions)
  - [Delivery Integration and Execution State](#delivery-integration-and-execution-state)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [References](#references)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

The [PlacementDecision API](../5313-placement-decision-api/README.md) identifies
the selected `ClusterProfile` objects. It does not say when each
cluster should receive an update. This KEP proposes a namespaced
`PlacementRolloutStrategy` resource that a delivery controller can use to divide
those selected clusters into stages, limit concurrent updates, and gate progress
on checks, pauses, or approvals.

A strategy describes progression independently of a workload or a particular
placement decision. A delivery resource identifies both the decision to use and
the strategy to follow. Its controller performs the updates and owns execution
state, including target results and approval decisions.

## Motivation

Consider a decision that selects one canary cluster, two staging clusters, and
twenty production clusters. A delivery controller can read that target set, but
the decision alone cannot express "update the canary first, verify it, then
update staging, then update production in groups of two after approval." Different
delivery controllers currently need their own configuration for this progression.
An operator cannot express the same rollout policy once for different workloads
that use the same cluster inventory and placement output.

### Goals

- Define reusable rollout stages over the `ClusterProfile` references in a
  `PlacementDecision`.
- Select stage targets by explicit reference, `ClusterProfile` labels, or the
  remaining selected clusters. Detect unintended overlap and missing update
  coverage.
- Support ordered and parallel stages, sequential partitions within a stage,
  per-stage concurrency and failure budgets, and readiness deadlines.
- Define when per-cluster and per-partition checks, timed pauses, and approvals
  block progression. Allow a referenced check to receive different inputs for
  each invocation.
- Let delivery controllers integrate the common strategy.

### Non-Goals

- Define how a placement producer selects clusters, or change `ClusterProfile`
  or `PlacementDecision`.
- Define a workload, release, execution, approval-request, or check-template API.
- Standardize payload installation, traffic shifting, rollback, or the health
  condition of a particular workload.
- Require every delivery controller to support every extension executor or
  check-template kind.

## Proposal

A `PlacementRolloutStrategy` contains a directed acyclic graph of stages. Each
stage selects targets from the placement decision, optionally divides them into
sequential partitions, and runs its steps for each partition. A stage starts
after all stages in `dependsOn` complete successfully. Stages with no dependency
path between them may run at the same time.

The common step types are `Update`, `Verify`, `Pause`, and `Approval`. An
`Extension` step reserves a position for an action whose behavior is defined by
the delivery controller.

### Users and Resources

| User | Reads | Creates or owns | Decision |
| --- | --- | --- | --- |
| Placement producer | `ClusterProfile` inventory | `PlacementDecision` slices | Which clusters are selected |
| Strategy author | Cluster labels and delivery requirements | `PlacementRolloutStrategy` | Target groups, order, budgets, and gates |
| Delivery controller | Decision slices, strategy, payload, and referenced templates | Delivery objects and execution state | How and when to update each selected cluster |
| Approver or approval service | Pending gate in execution state | A decision through the delivery controller's interface | Whether one execution may continue |

### Risks and Mitigations

* Unintended selector match: Selectors are intersected with the decision target
  set. Resolution reports the final targets before updates begin.
* Uncovered or overlapping targets: A selected target with no update stage, or
  in incompatible stages, fails resolution by default. Explicit policies allow
  intended exclusions and ordered repeats.
* Total concurrency across parallel stages: Each stage has its own `maxUpdate`.
  Delivery status exposes the active count across stages.
* Changes during an execution: The delivery controller freezes a consistent view
  of placement, labels, and templates for each execution.
* Unsupported gate or extension: The controller rejects the strategy for that
  delivery before updating a target.
* Excess data access by an expression or check: Expressions see only frozen
  `ClusterProfile` data. Template execution uses separately granted credentials.
* Check failure after an update: The strategy stops further scheduling as
  specified.

## Design Details

### API Types

The API group is `multicluster.x-k8s.io`, version `v1alpha1`.
`PlacementRolloutStrategy` is namespace scoped. The following types illustrate
the wire format; API review may refine Go helper types and markers while
preserving the specified fields and behavior.

```go
type PlacementRolloutStrategy struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec PlacementRolloutStrategySpec `json:"spec"`
}

type PlacementRolloutStrategySpec struct {
    // Fail (default) or Exclude selected targets with no update stage.
    UnmatchedPolicy UnmatchedPolicy `json:"unmatchedPolicy,omitempty"`
    Stages []PlacementRolloutStage `json:"stages"`
}

type PlacementRolloutStage struct {
    Name string `json:"name"`
    DependsOn []string `json:"dependsOn,omitempty"`
    Targets StageTargets `json:"targets"`
    AllowRepeatedTargets bool `json:"allowRepeatedTargets,omitempty"`
    AllowEmpty bool `json:"allowEmpty,omitempty"`
    Partition *StagePartition `json:"partition,omitempty"`
    MaxUpdate *intstr.IntOrString `json:"maxUpdate,omitempty"`
    MaxFailures *intstr.IntOrString `json:"maxFailures,omitempty"`
    Steps []PlacementRolloutStep `json:"steps"`
}

type StageTargets struct {
    // Exactly one selection mode is set.
    ClusterProfileSelector *metav1.LabelSelector `json:"clusterProfileSelector,omitempty"`
    ClusterProfileRefs []corev1.ObjectReference `json:"clusterProfileRefs,omitempty"`
    Remaining bool `json:"remaining,omitempty"`
}

type StagePartition struct {
    // Maximum targets per sequential partition.
    MaxClusters intstr.IntOrString `json:"maxClusters"`
}

type PlacementRolloutStep struct {
    Type PlacementRolloutStepType `json:"type"`

    // Update
    ProgressDeadlineSeconds *int32 `json:"progressDeadlineSeconds,omitempty"`
    MinSuccessSeconds *int32 `json:"minSuccessSeconds,omitempty"`

    // Verify
    TemplateRef *LocalResourceReference `json:"templateRef,omitempty"`
    Scope VerificationScope `json:"scope,omitempty"`
    RunAt VerificationLocation `json:"runAt,omitempty"`
    Inputs map[string]apiextensionsv1.JSON `json:"inputs,omitempty"`
    TimeoutSeconds *int32 `json:"timeoutSeconds,omitempty"`

    // Pause
    DurationSeconds *int32 `json:"durationSeconds,omitempty"`

    // Extension
    ExecutorName string `json:"executorName,omitempty"`
    ParametersRef *LocalResourceReference `json:"parametersRef,omitempty"`
}

type PlacementRolloutStepType string

const (
    UpdateStepType PlacementRolloutStepType = "Update"
    VerifyStepType PlacementRolloutStepType = "Verify"
    PauseStepType PlacementRolloutStepType = "Pause"
    ApprovalStepType PlacementRolloutStepType = "Approval"
    ExtensionStepType PlacementRolloutStepType = "Extension"
)

type LocalResourceReference struct {
    APIVersion string `json:"apiVersion"`
    Kind string `json:"kind"`
    Name string `json:"name"`
}
```

The strategy has no shared `status`. Several deliveries may use it
simultaneously, so execution state belongs to each delivery.

### Resolving Targets

The delivery controller first resolves all slices of one logical
`PlacementDecision` as described by [KEP-5313](../5313-placement-decision-api/README.md).
Repeated `clusterProfileRef` values represent one target. Target identity is
the `ClusterProfile` namespace/name pair. The controller obtains the selected
`ClusterProfile` objects and evaluates selectors against their labels.

`clusterProfileRefs` entries MUST identify
`multicluster.x-k8s.io/v1alpha1` `ClusterProfile` objects by API version, kind,
namespace, and name. Duplicate references within a stage are invalid. A missing
referenced or decision-selected `ClusterProfile` prevents resolution.

`remaining: true` selects targets not matched by *any* explicit-reference or
selector stage. It is calculated over the full strategy and may appear in at
most one stage.

The following rules apply to the resolved stage targets:

- A target in more than one stage is an error unless those stages are ordered
  by dependencies. For each pair containing the same target, the dependent
  stage MUST set `allowRepeatedTargets: true` and depend directly or
  transitively on the other stage. A repeat causes another payload update only
  if the dependent stage contains an `Update` step.
- With the default `unmatchedPolicy: Fail`, every decision target MUST be
  selected by at least one stage containing `Update`. `Exclude` allows
  unmatched targets to receive no update.
- A stage with no resolved targets is an error unless `allowEmpty: true`. An
  allowed empty stage completes without running its steps.

Target selection, the `ClusterProfile` data used by selectors and expressions,
stage assignment, the effective strategy, and referenced check templates are
frozen for one execution. The controller uses a consistent view of decision
slices. If it cannot establish a complete target set, it waits. Its delivery
API defines whether a later placement or strategy change finishes, cancels, or
starts a new execution and reports that choice in delivery status.

### Stages, Partitions, and Update Budgets

Stage names are unique DNS labels. `dependsOn` names stages in the same
strategy. Duplicate dependencies, unknown names, self-dependencies, and cycles
are invalid.

For partitioning, sort a stage's frozen targets by `namespace/name` and split
the list into consecutive groups of at most `partition.maxClusters`. With no
`partition`, the entire stage is one partition. A percentage is applied to the
stage's target count, rounded up, with a minimum of one for a nonempty stage.
The sort provides deterministic grouping. Every step in one partition finishes
before the next partition starts.

`maxUpdate` limits updates in flight within a stage. A percentage is calculated
as for `partition.maxClusters`. A count above the stage size acts as the stage
size. An update holds a slot from its first attempt until the target reaches the
delivery controller's ready state for `minSuccessSeconds` or fails.

An `Update` step may set `progressDeadlineSeconds`. The deadline runs from the
first attempt until the target meets the readiness requirement; expiry is a
target failure. The controller defines the payload's ready state and retry
policy.

`maxFailures` is the maximum number of failed targets tolerated by a stage.
An integer is a count; a percentage of the stage target count is rounded down.
Each target that fails its update or a per-cluster check counts once, even after
retries or failures in multiple checks. Later per-cluster steps skip that failed
target. Partition-level steps may still inspect the partition and its failed
targets. A partition check failure, rejected approval, or failed extension fails
the stage regardless of the target failure budget.

Exceeding `maxFailures` fails the stage and the execution. Its dependents
cannot start, and the execution stops scheduling new updates in every branch.
The delivery controller defines how already-running operations end and records
their outcomes. A stage that stays within its failure budget may complete and
unblock dependents; its failed targets remain visible in execution results.

### Steps and Verification

Steps run in their listed order for each partition.

`Update` asks the delivery controller to perform its normal payload update on
the partition's eligible targets and wait for their readiness outcomes.
`Verify` invokes a referenced check template. The same template may be used in
several steps or stages with different inputs.

| `Verify` setting | Meaning |
| --- | --- |
| `scope: PerCluster` (default) | Run separately for each eligible target in the current partition. |
| `scope: Partition` | Run once for the current partition, including its target results. |
| `runAt: ControlPlane` (default) | Run in the management environment. |
| `runAt: TargetCluster` | Run in the target cluster; valid only for `PerCluster`. |
| `timeoutSeconds` | Fail the invocation if it does not complete within the specified time. |

Each invocation is identified by its delivery execution, stage, partition, step
position, and, for `PerCluster`, target namespace/name. Retries preserve that
identity. Inputs and results belong to the invocation. If a controller combines
several invocations in one runtime resource, it MUST preserve their individual
inputs and outcomes.

`inputs` maps nonempty names to JSON values. Literal values preserve their JSON
types. String leaves may contain `${...}` expressions evaluated with Common
Expression Language (CEL). If an entire string is one expression, its
JSON-compatible result replaces the string; an expression embedded in other
text MUST return a string. The CEL expression `${"${VAR}"}` emits literal
`${VAR}` text. The only read-only variables are:

| Variable | Scope | Value |
| --- | --- | --- |
| `target` | `PerCluster` | The current target's frozen `ClusterProfile` object |
| `targets` | Both scopes | The current partition's frozen `ClusterProfile` objects |

For example, a per-cluster check can receive a label with a fallback and a
numeric HTTP status:

```yaml
inputs:
  region: '${"region" in target.metadata.labels ? target.metadata.labels["region"] : "global"}'
  expectedStatusCode: 200
  targetCount: ${size(targets)}
```

Expressions can inspect the supplied objects, including metadata and
`status.properties`. Missing label and annotation maps are exposed as empty
maps. The controller evaluates every invocation's inputs against the frozen
`ClusterProfile` data before the first update. Missing keys, invalid types, or
other evaluation errors prevent the execution from starting.

Before updating a target, the controller resolves referenced templates. It
validates template kind, input names and types, and execution location. The
template API, input mapping, result format, credentials, and cleanup belong to
the delivery integration.

`Pause` waits for a positive `durationSeconds` after the previous step.
`Approval` waits for an explicit approved or rejected decision for the current
partition. It does not occupy an update slot. The delivery controller owns the
interface for submitting a decision and records the actor and outcome. The
decision is bound to the execution, desired payload revision, stage, partition,
and step position.

`Extension` names an executor supported by the delivery controller and may
reference a local parameters object. The controller defines the action and its
effect on readiness and failures. An extension that starts target updates
observes the same stage `maxUpdate` budget as `Update`.

### Validation and Access

The CRD schema validates required fields, enums, bounds, and fields allowed for
each step type. Admission validates graph properties that the structural schema
cannot check. Decision-dependent checks and delivery capabilities are validated
by the delivery controller before execution.

| Field | Default | Accepted value |
| --- | --- | --- |
| `targets` | None | Exactly one of selector, nonempty references, or `remaining: true` |
| `partition.maxClusters` | Whole stage | Positive integer or `1%`–`100%` |
| `maxUpdate` | `100%` | Positive integer or `1%`–`100%` |
| `maxFailures` | `0` | Nonnegative integer or `0%`–`100%` |
| `steps` | None | Nonempty list; at most one `Update` |
| `minSuccessSeconds` on `Update` | `0` | Nonnegative integer |
| `progressDeadlineSeconds` on `Update` | No deadline | Positive integer when set |
| `templateRef` on `Verify` | None | Required local resource reference |
| `timeoutSeconds` on `Verify` | No timeout | Positive integer when set |
| `durationSeconds` on `Pause` | None | Required positive integer |
| `Approval` | Pending until a decision | No type-specific fields |
| `executorName` on `Extension` | None | Required nonempty name |

If both update durations are set, `minSuccessSeconds` cannot exceed
`progressDeadlineSeconds`. Admission scans string leaves for expressions and
rejects invalid syntax, unknown variables, and statically detectable type
errors. Evaluation uses bounded cost.

A namespaced delivery resource references a strategy in its own namespace.
Template and parameter references are also local to the strategy's namespace.
The delivery integration determines which decision namespace it may read. A
cluster-scoped delivery API MUST define its own namespace binding. Strategy
write permission does not grant access to a target cluster.

### API Example: Canary, Staging, Production

This example uses one named canary, staging clusters identified by inventory
labels, and all other clusters selected by the decision for production.
`JobCheckTemplate` and `FleetCheckTemplate` are illustrative template kinds
supplied by the delivery integration.

```yaml
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: PlacementRolloutStrategy
metadata:
  name: application-rollout
  namespace: applications
spec:
  stages:
  - name: canary
    targets:
      clusterProfileRefs:
      - apiVersion: multicluster.x-k8s.io/v1alpha1
        kind: ClusterProfile
        namespace: fleet
        name: canary-1
    maxUpdate: 1
    steps:
    - type: Update
      progressDeadlineSeconds: 600
      minSuccessSeconds: 60
    - type: Verify
      templateRef:
        apiVersion: checks.example.io/v1alpha1
        kind: JobCheckTemplate
        name: smoke-test
      runAt: TargetCluster
      inputs:
        clusterName: ${target.metadata.name}
        requestPath: /healthz
  - name: staging
    dependsOn: [canary]
    targets:
      clusterProfileSelector:
        matchLabels:
          environment: staging
    partition:
      maxClusters: 2
    maxUpdate: 2
    steps:
    - type: Update
    - type: Verify
      templateRef:
        apiVersion: checks.example.io/v1alpha1
        kind: JobCheckTemplate
        name: smoke-test
      runAt: TargetCluster
      inputs:
        clusterName: ${target.metadata.name}
        requestPath: /healthz
  - name: production
    dependsOn: [staging]
    targets:
      remaining: true
    partition:
      maxClusters: 2
    maxUpdate: 2
    steps:
    - type: Verify
      templateRef:
        apiVersion: checks.example.io/v1alpha1
        kind: FleetCheckTemplate
        name: production-preflight
      scope: Partition
      inputs:
        targetCount: ${size(targets)}
    - type: Approval
    - type: Update
      progressDeadlineSeconds: 900
    - type: Verify
      templateRef:
        apiVersion: checks.example.io/v1alpha1
        kind: JobCheckTemplate
        name: smoke-test
      runAt: TargetCluster
      inputs:
        clusterName: ${target.metadata.name}
        requestPath: /healthz
    - type: Verify
      templateRef:
        apiVersion: checks.example.io/v1alpha1
        kind: JobCheckTemplate
        name: smoke-test
      runAt: TargetCluster
      inputs:
        clusterName: ${target.metadata.name}
        requestPath: /checkout
```

Production approval is requested for each partition of two.

### API Example: Parallel Regions

Independent regional stages can start together. A final stage waits for both
and checks their combined targets without updating them again.

```yaml
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: PlacementRolloutStrategy
metadata:
  name: regional-rollout
  namespace: applications
spec:
  stages:
  - name: us
    targets:
      clusterProfileSelector:
        matchLabels:
          region: us
    maxUpdate: 6
    steps:
    - type: Update
  - name: eu
    targets:
      clusterProfileSelector:
        matchLabels:
          region: eu
    maxUpdate: 6
    steps:
    - type: Update
  - name: global-check
    dependsOn: [us, eu]
    targets:
      clusterProfileSelector:
        matchExpressions:
        - key: region
          operator: In
          values: [us, eu]
    allowRepeatedTargets: true
    steps:
    - type: Verify
      templateRef:
        apiVersion: checks.example.io/v1alpha1
        kind: FleetCheckTemplate
        name: regional-health
      scope: Partition
```

The regional update limits can permit twelve updates at once. A decision target
outside the two regions fails resolution under the default unmatched policy.

### Delivery Integration and Execution State

A delivery resource MUST identify the complete logical decision and the
strategy. One possible integration looks like this:

```yaml
apiVersion: delivery.example.io/v1alpha1
kind: FleetDelivery
metadata:
  name: storefront
  namespace: applications
spec:
  placementDecisionKey: storefront-targets
  rolloutStrategyRef:
    name: application-rollout
  payloadRef:
    name: storefront-v2
```

In this example the placement producer labels every decision slice in
`applications` with `multicluster.x-k8s.io/decision-key=storefront-targets`. The
delivery controller collects those slices, resolves the strategy, and binds the
desired payload revision to one execution.

The controller records the resolved target set, stage and partition assignments,
effective templates and inputs, progress, failures, and approvals in its own
execution state. It exposes a condition when resolution fails or a required
capability is unsupported. It MUST NOT ignore an unknown standard step or gate.
Its status SHOULD make active partitions, target readiness, check results,
pending approvals, and terminal causes visible to users.

### Test Plan

API validation tests cover target selection modes, duplicate references,
required and forbidden step fields, quantity bounds, duplicate stage names,
unknown dependencies, cycles, and invalid check scope/location pairs. Expression
tests cover literal JSON types, nested inputs, standalone and embedded
expressions, escaping, unknown variables, missing keys, optional-key fallbacks,
and evaluation cost limits.

Resolution tests cover sliced decisions, selector intersection with the decision,
missing `ClusterProfile` objects, remaining targets, empty stages, unmatched
targets, ordered repeats, deterministic partitions, and percentage rounding.
They also cover a decision or label change during an execution.

Delivery integration tests cover serial and parallel stages, independent update
budgets, target readiness and deadlines, tolerated target failures, partition
failures, checks before and after updates, repeated templates with independent
inputs and results, frozen template contents, pauses, current and stale approval
decisions, unsupported executors, and status. Target-cluster checks verify
that the intended target and credentials are used. These tests belong to future
API and delivery implementations; the KEP does not add runtime code.

### Graduation Criteria

#### Alpha

- API types and a CRD schema implement the specified fields, validation, and
  defaults.
- At least one delivery integration resolves complete decisions, enforces
  selection and progression rules, and reports unsupported steps before updates.
- Behavior tests cover validation, target resolution, graph order, and failures.

#### Beta

- Two independent delivery integrations use the same standard stage, update,
  pause, approval, and verification examples.
- Conversion and version-skew behavior are documented and tested.
- Scale and observability tests cover large decisions, partitions, parallel
  stages, and concurrent deliveries using one strategy.

#### GA

- Implementation feedback has resolved ambiguous defaults and portability gaps.
- Conformance coverage exercises portable fields and documented error cases.
- User documentation and production readiness review are complete.

### Upgrade / Downgrade Strategy

The new CRD does not change a delivery that has no strategy reference. After a
downgrade, a controller MUST reject any strategy field or gate it cannot honor
before starting an execution. Its existing execution state determines what
happens to work already in flight.

### Version Skew Strategy

The CRD follows normal Kubernetes API versioning. A delivery controller declares
which strategy versions, template kinds, and extension executors it supports.
Conversion MUST preserve gates and their meaning. A strategy shared by
controllers on different versions can use only capabilities supported by all
intended controllers.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

Installing the strategy CRD permits authors to create strategies but starts no
delivery. A delivery controller uses a strategy only when its delivery resource
references one. The controller's own API and execution policy govern disabling
new executions and handling active payloads.

### Rollout, Upgrade and Rollback Planning

Delivery controllers resolve the complete strategy, target set, templates, and
required capabilities before changing a target. A strategy edit affects new
executions; an active execution uses its frozen version. The controller
defines what happens to active work when its integration is upgraded or disabled
and whether it can reverse an already-applied payload.

### Monitoring Requirements

The strategy has no status or metrics. The delivery integration reports current
stage and partition, in-flight updates, readiness, check results, failure count,
pending approval, and terminal cause through its own status and metrics. A
missing or unsupported template is distinguishable from a check that ran and
failed.

### Dependencies

Target resolution uses `ClusterProfile` and `PlacementDecision`. A `Verify`
step additionally needs a template kind and executor supported by the delivery
controller. An `Extension` needs its named executor. Neither dependency is
created by this KEP.

### Scalability

Selectors and percentage partitions keep a strategy smaller than its selected
target set. Frozen targets and results belong to delivery execution storage.
Controllers SHOULD cache inventory and decision slices, bound remote check
concurrency, and avoid scanning unrelated namespaces on each update. CEL
compilation and evaluation are cost bounded, and inputs are resolved once per
invocation.

### Troubleshooting

Admission reports invalid strategy structure. Delivery status reports
decision-dependent errors such as incomplete slices, missing profiles,
uncovered or overlapping targets, empty stages, input evaluation errors, and
unsupported templates or executors. During an execution it distinguishes
waiting for capacity, readiness, verification, pause, approval, and terminal
failure.

## Implementation History

No implementation milestones have been reached.

## Drawbacks

An operator must manage another resource and reference. Reusing a strategy also
means an edit may affect several future deliveries. Common progression rules
cannot guarantee identical payload readiness, check results, or rollback across
different delivery controllers; those behaviors remain visible in each
controller's API and execution status.

## Alternatives

**Put rollout controls in `PlacementDecision`.** That would make the placement
producer responsible for delivery order, checks, and approvals, even though it
does not update the payload.

**Put the full strategy in each delivery resource.** This avoids a separate
reference for one controller but requires each delivery API to define its own
stage and gate schema and prevents sharing one policy across workloads.

**Define a common execution resource.** Such a resource would need shared
payload identity, update, health, approval, and rollback semantics. This KEP
defines the reusable policy and leaves those workload-dependent behaviors
with each delivery controller.

**Use an ordered list of stages.** A list expresses a canary followed by
staging and production, but cannot express independent regional branches
followed by a common check. Stage dependencies cover both patterns.

## References

- [Cluster Inventory API (KEP-4322)](../4322-cluster-inventory/README.md)
- [PlacementDecision API (KEP-5313)](../5313-placement-decision-api/README.md)
