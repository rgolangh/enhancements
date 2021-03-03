---
title: node-healthcheck-controller
authors:
  - "@rgolangh"
reviewers:
  - TBD
  - "@abeekof"
  - "@nir1s"
approvers:
  - TBD
creation-date: 2021-03-02
last-updated: 2021-03-02
status: implementable
see-also:
  - "/enhancements/this-other-neat-thing.md"
replaces:
  - "/enhancements/that-less-than-great-idea.md"
superseded-by:
  - "/enhancements/our-past-effort.md"
---

# Node health checking

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift/docs


## Summary


Enable opt in for automated node health checking and create objects
for unhealthy nodes, to be handled later by external controllers
that comply to the external [api].

## Motivation

Enable non machine-managed clusters to remedy, and/or restore capacity.

### Goals

- Feature parity with Machine Health Check controller

- Providing a default strategy combined with [poison-pill].

- Endorse usage of using external [remediation api]

- Allow defining different unhealthy criteria by node selectors.

- Provide defaults health checks and default remedy custom resource creation.

- Code share with Machine Health Check controller relevant parts. (TBD not sure about this)

- Long term - Make Machine Health Check controller to implement the external remediation api and 
drop the health checking parts


### Non-Goals

- Coordination of remedy operation - operators/controllers implementors 
expected to track the [node maintenance lease proposal]

- Recording long-term stable history of all health-check failures or remediation
  actions.

## Proposal

- As an admin of a cluster I want the cluster to detect Node failures and
 initiate recovery, so that workloads and volumes with at-most-one semantics
  can be run/mounted elsewhere.

- As an admin of a cluster I want the cluster to detect Node failures and
 initiate recovery, so that we can attempt to restore cluster capacity.

- As an OCP engineer, I want the NHC to consume the same logic for detecting
 failures, to avoid code duplication and provide a consistent experience
  compared to MHC.

- As an OCP engineer, I want the NHC to use the same interface for initiating
 recovery, so that we can reuse existing mechanisms (like [poison pill]) and
  avoid code duplication.

To fulfil the above we create a [Node Health Check custom resource (NCR)](anchor todo) that specifies:
1. health criteria: list of criterias and selector
2. template: a template of an [external custom resource (RCR)](link todo)
 to create in response to a match.

### Unhealthy criteria
A node target is unhealthy when the node meets the unhealthy node conditions criteria defined.
If any of those criterias are met for longer than given timeouts the controller
creates an RCR according to the template in the NCR.
Timeouts will be defined by the controller and are configurable using cli args.

### Implementation Details

#### NodeHealthCheck CRD
- Enable watching a node based on a label selector. Exclude masters is always the first selector internally.

- Enable defining an unhealthy node criteria (based on a list of node conditions).
- Enable setting a threshold of unhealthy nodes. If the current number is at or above this threshold no further will take place. This can be expressed as an int or as a percentage of the total targets in the pool.

E.g:
- Create a remedy CRD when node is `ready=false` or `ready=Unknown` condition for more than 10m.
- I want to temporary short-circuit if the 40% or more of the targets of this pool are unhealthy at the same time.


```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
kind: NodeHealthCheck
metadata:
  name: example
  # this CRD is cluster-scope, namespace is not needed  
spec:
  selector:
    matchLabels:
      role: worker
  unhealthyConditions:
  - type:    "Ready"
    status:  "Unknown"
    timeout: "300s"
  - type:    "Ready"
    status:  "False"
    timeout: "300s"
  maxUnhealthy: "40%"
status:
  currentHealthy: 5
  expectedMachines: 5
```

#### NodeHealthCheck controller
Watch:
- Watch NodeHealthCheck resources
- Watch nodes with an event handler e.g controller runtime `EnqueueRequestsFromMapFunc` which returns NodeHealthCheck resources.

Reconcile:
- Periodically enqueue a reconcile request
- Fetch all nodes targets. E.g:

```go
type target struct {
  Node    *corev1.Node
  NHC     capi.NodeHealthCheck
}
```

- Calculate the number of unhealthy targets.
- Compare current number against `maxUnhealthy` threshold and temporary short
 circuits if the threshold is met.
- Creat remedy objects for unhealthy targets according to the template in NHC object

### Notes/Constraints

The functionality of this controller have a lot in common with the Machine Health
Check controller expect that its creating objects and not 
performing any actions per se. When deploying this controller make
sure 1. you have a controller implementing the API, e.g [poison pill]
2. the machine health check controller is not running, or in case of a mixed 
 the node health checks selector avoid targeting nodes with machines.


### Risks and Mitigations

#### Remediation Interaction with ClusterAutoScalar 

Misconfiguration of nodes provisioning can cause nodes to fails to get operational.
Remediation is blind to such problems and may interfere with diagnosing the problem.
The backoff mechanism mentioned in MHC may be suited for Machines, because they
have the MachineSet name that can be recorded and tracked. Without the machine API
our knowledge of the nodes could not be handling in such a generic way. There is
simply no generic way to record a node configuration origin in reliable way without
putting provider specific annotations.
Potentially the NHC resource could contain label selectors to track
repeating remediation and mark them as [incrementally back-off](https://issues.redhat.com/browse/OCPCLOUD-800). e.g

Suggested backoff specificiation under the NHC resource may look like this:
```yaml
kind: NodeHealthCheck
...
spec:
  selector:
     ...
  # optional
  backoff:
    type: exponential
    limit: 10m
    selector:
      matchLabels:
        topology.kubernetes.io/zone=us-east-1c
      matchExpressions:
        - {key: kubernetes.io/hostname, operator: In, values: [ip-172-20-114-199.ec2.internal]}: 
...
```
Caveat of this approach is that we need to take care of conflicting or overlapping backoff definition.
Setting global catchall backoff strategy is impossible or just coarse without specifying a selector (how do we know what to backoff from?)

#### Multiple remediation provider deployed

Multiple providers could act on remediation objects. On non heterogeneous cluster
this may even make sense. It is not the goal of this controller to coordinate
those, but providers will need disparate set of nodes to handle based on labels,
or operator on a node if they can hold a [maintenance lease][node maintenance lease proposal].


# [rgolan] FROM HERE ON THIS IS ALL IN PROGRESS #

## Design Details

### Open Questions [optional]

This is where to call out areas of the design that require closure before deciding
to implement the design.  For instance,
 > 1. This requires exposing previously private resources which contain sensitive
  information.  Can we do this?

### Test Plan

**Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?
- What additional testing is necessary to support managed OpenShift service-based offerings?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).

### Graduation Criteria

**Note:** *Section not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, or as something else. Initial proposal
should keep this high-level with a focus on what signals will be looked at to
determine graduation.

Consider the following in developing the graduation criteria for this
enhancement:

- Maturity levels
  - [`alpha`, `beta`, `stable` in upstream Kubernetes][maturity-levels]
  - `Dev Preview`, `Tech Preview`, `GA` in OpenShift
- [Deprecation policy][deprecation-policy]

Clearly define what graduation means by either linking to the [API doc definition](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning),
or by redefining what graduation means.

In general, we try to use the same stages (alpha, beta, GA), regardless how the functionality is accessed.

[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/

**Examples**: These are generalized examples to consider, in addition
to the aforementioned [maturity levels][maturity-levels].

#### Dev Preview -> Tech Preview

- Ability to utilize the enhancement end to end
- End user documentation, relative API stability
- Sufficient test coverage
- Gather feedback from users rather than just developers
- Enumerate service level indicators (SLIs), expose SLIs as metrics
- Write symptoms-based alerts for the component(s)

#### Tech Preview -> GA

- More testing (upgrade, downgrade, scale)
- Sufficient time for feedback
- Available by default
- Backhaul SLI telemetry
- Document SLOs for the component
- Conduct load testing

**For non-optional features moving to GA, the graduation criteria must include
end to end tests.**

#### Removing a deprecated feature

- Announce deprecation and support policy of the existing feature
- Deprecate the feature

### Upgrade / Downgrade Strategy

If applicable, how will the component be upgraded and downgraded? Make sure this
is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to keep previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to make use of the enhancement?

Upgrade expectations:
- Each component should remain available for user requests and
  workloads during upgrades. Ensure the components leverage best practices in handling [voluntary disruption](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/). Any exception to this should be
  identified and discussed here.
- Micro version upgrades - users should be able to skip forward versions within a
  minor release stream without being required to pass through intermediate
  versions - i.e. `x.y.N->x.y.N+2` should work without requiring `x.y.N->x.y.N+1`
  as an intermediate step.
- Minor version upgrades - you only need to support `x.N->x.N+1` upgrade
  steps. So, for example, it is acceptable to require a user running 4.3 to
  upgrade to 4.5 with a `4.3->4.4` step followed by a `4.4->4.5` step.
- While an upgrade is in progress, new component versions should
  continue to operate correctly in concert with older component
  versions (aka "version skew"). For example, if a node is down, and
  an operator is rolling out a daemonset, the old and new daemonset
  pods must continue to work correctly even while the cluster remains
  in this partially upgraded state for some time.

Downgrade expectations:
- If an `N->N+1` upgrade fails mid-way through, or if the `N+1` cluster is
  misbehaving, it should be possible for the user to rollback to `N`. It is
  acceptable to require some documented manual steps in order to fully restore
  the downgraded cluster to its previous state. Examples of acceptable steps
  include:
  - Deleting any CVO-managed resources added by the new version. The
    CVO does not currently delete resources that no longer exist in
    the target version.

### Version Skew Strategy

How will the component handle version skew with other components?
What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- During an upgrade, we will always have skew among components, how will this impact your work?
- Does this enhancement involve coordinating behavior in the control plane and
  in the kubelet? How does an n-2 kubelet without this feature available behave
  when this feature is used?
- Will any other components on the node change? For example, changes to CSI, CRI
  or CNI may require updating that component before the kubelet.

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.

## Drawbacks

The idea is to find the best form of an argument why this enhancement should _not_ be implemented.

## Alternatives

Similar to the `Drawbacks` section the `Alternatives` section is used to
highlight and record other possible approaches to delivering the value proposed
by an enhancement.

## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new
subproject, repos requested, github details, and/or testing infrastructure.

Listing these here allows the community to get the process for these resources
started right away.


[poison pill]: https://github.com/poison-pill/poison-pill
[node maintenance lease proposal]: https://github.com/kubernetes/enhancements/pull/1411/
[remediaion API]: 
