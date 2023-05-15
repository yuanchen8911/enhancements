# KEP-3902:  Decouple Taint-based Pod Eviction from Node Lifecycle Controller 
<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories (Optional)](#user-stories-optional)
    - [Story](#story)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Proposed Controllers](#proposed-controllers)
  - [Implementation](#implementation)
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
- [Infrastructure Needed (Optional)](#infrastructure-needed-optional)
- [Note](#note)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ]  (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements](https://git.k8s.io/enhancements) (not the initial KEP PR)
- [ ]  (R) KEP approvers have approved the KEP status as `implementable`
- [ ]  (R) Design details are appropriately documented
- [ ]  (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
    - [ ]  e2e Tests for all Beta API Operations (endpoints)
    - [ ]  (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
    - [ ]  (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ]  (R) Graduation criteria is in place
    - [ ]  (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
- [ ]  (R) Production readiness review completed
- [ ]  (R) Production readiness review approved
- [ ]  "Implementation History" section is up-to-date for milestone
- [ ]  User-facing documentation has been created in [kubernetes/website](https://git.k8s.io/website), for publication to [kubernetes.io](https://kubernetes.io/)
- [ ]  Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

## Summary

In Kubernetes, `NodeLifecycleController` applies predefined `NoExecute` taints (e.g., `Unreachable`, `NotReady`) when the nodes are determined to be unhealthy. After the nodes get tainted, `TaintManager` does its due diligence to start deleting running pods on those nodes based on `NoExecute` taints, which can be added by anyone.

In this KEP, we propose to decouple `TaintManager` that performs taint-based pod eviction from `NodeLifecycleController` and make them two separate controllers: `NodeLifecycleController` to add taints to unhealthy nodes and `TaintManager` to perform pod deletion on nodes tainted with NoExecute effect.

This separation not only improves code organization but also makes it easier to improve `TaintManager` or build custom `TaintManager`.

## Motivation
`NodeLifecycleController` combines two independent functions: 
   * Adding a pre-defined set of `NoExecute` taints to nodes based on the node conditions 
   * Performing pod eviction on `NoExecute` taints

Splitting the `NodeLifecycleController` based on the above functionalities will help to disentagle code and make future extensions to either component manageable.

### Goals

* Move taint-based eviction implementation out of `NodeLifecycleController` into a separate and independent taint-manager to enhance separation of concerns, maintainability. This separation also enables cluster-admins to build custom `TaintManager`s and use them in their clusters.

### Non-Goals

* Improve and change the APIs to support extension and customization of the existing `TaintManager`.
* Actual extension and improvement of the current `TaintManager`.

## Proposal

### User Stories (Optional)

#### Story

While this split is more of a kubernetes developer focused change, an unintended positive effect of this change is that it allows cluster-admins to develop custom `TaintManager`s and disable the default `TaintManager`. 

The default `TaintManager`, because of the implementation rigidity, may not be suitable for every cluster operator. We except users to come up with custom `TaintManager`s and this KEP may exacerbate the problem of `bring your own taint manager`. However, discussing the use-cases for extending `TaintManager` or custom `TaintManger`s is beyond the scope of this KEP. 

### Notes/Constraints/Caveats (Optional)

### Risks and Mitigations

## Design Details

In version 1.27 and prior of Kubernetes, `NoExecuteTaintManager` is part of `NodeLifecycleController`. It’s created and run if `enable-taint-manager` flag is true.  The flag is deprecated and will be removed in kubernetes 1.27. 

<p align="center"><image src="current-nodelifecycle-controller.png" width=50%></p>

### Proposed Controllers

The proposed design refactors `NodeLifecycleController` into two independent controllers, which are managed by `kube-controller-manager`.

1. `NodeLifecycleController` monitors node health and adds `NotReady` and `Unreachable` taints to nodes 
2. `NoExecuteTaintManager` watches for node and pod updates,  and performs `NoExecute` taint based pod eviction

The existing  kube-controller-manager flag `--controllers` will be used to optionally disable the `NoExecuteTaintManager`.

<p align="center"><image src="new-controllers.png" width=60%></p>

### Implementation

A new `NodeLifecycleManager` is implemented by removing taint-manager related code from `controller/nodelifecycle/node_lifecycle_controller.go`.

A new `NoExecuteTaintManager` is created a first-class controller managed by` kube-controller-manager`. It’s implementation is based on the current taint-manager  `controller/nodelifecycle/taint-manager.go`.   

The creation and startup of the default taint-manager is performed by `kube-controller-manager` . 

```
func startNoExecuteTaintManager(ctx context.Context, controllerContext ControllerContext) (controller.Interface, bool, error) {`
    if !controllerContext.ComponentConfig.NoExecuteTaintManager.EnableNoExecuteTaintManager {
        return nil, false, nil
    }

taintManager, err = taintmanager.NewNoExecuteTaintManager(ctx, kubeClient, nc.podLister, nc.nodeLister, nc.getPodsAssignedToNode)
        nodeInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
            AddFunc: controllerutil.CreateAddNodeHandler(func(node *v1.Node) error {
                nc.taintManager.NodeUpdated(nil, node)
                return nil
            }),
            UpdateFunc: controllerutil.CreateUpdateNodeHandler(func(oldNode, newNode *v1.Node) error {
                nc.taintManager.NodeUpdated(oldNode, newNode)
                return nil
            }),
            DeleteFunc: controllerutil.CreateDeleteNodeHandler(func(node *v1.Node) error {
                nc.taintManager.NodeUpdated(node, nil)
                return nil
            }),
        })
    }
        if err != nil {
        return nil, true, fmt.Errorf("failed to start the taint manager: %v", err)
    }
  }  
```

### Test Plan

* Unit tests
    * Enable `TaintManager`
    * Disable `TaintManager`
    * Add `NoExecute` taints
    * Act on `NoExecute` to delete running pods
    * Respect taint toleration 
* Integration tests
    * Enable `TaintManager`, add `NoExecute` taints to evict the correct set of running pods on nodes 
    * Disable `TaintManager`, add `NoExecute` has no impact on running pods
* E2E tests 
    * Verify passing the existing E2E and conformance tests.

### Graduation Criteria

#### Alpha

* Support `kube-controller-manager` flag `controllers=-taint-manager` to disable the default taint-manager.
* Unit and integration tests completed and passed.

#### Beta

* Performance and scalability tests to verify there is non-negligible performance degradation in node taint-based eviction.
* Update documents to reflect the changes.

#### GA

* Fix all reported bugs if any.

### Upgrade / Downgrade Strategy

We’ll need to upgrade `kube-controller-manager` and` node-lifecycle-controller` and add a new controller `taint-manager`. Accordingly, a downgrade will revert to the previous version of `kube-controller-manager` and `node-lifecycle-controller` and remove the new added `taint-manager` . There should be no impact on the other components during upgrades and downgrades since user-exposed behavior should stay the same except a new flag to enable/disable the default `taint-manager`. This change should be clearly communicated to users.

### Version Skew Strategy

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

*This section must be completed when targeting alpha to a release.*

* **How can this feature be enabled / disabled in a live cluster?**
    * Enabled:  upgrade `kube-controller-manager` and` node-lifecycle-controller` and add a new `taint-manager` controller.
    * Disabled: downgrade will revert to the previous version of `kube-controller-manager` and `node-lifecycle-controller` and remove the new added `taint-manager` . 
* **Does enabling the feature change any default behavior?**
    * No, the default node-life-cycle controller and taint-manager behavior will stay the same.
* **Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?**
    * Yes, we will need to rack back `kube-controller-manager` and `node-lifecycle-controller`. 
* **What happens if we re-enable the feature if it was previously rolled back?**
    * Upgrade `kube-controller-manager` and `node-lifecycle-controller` and add the new `taint-manager`.
* **Are there any tests for feature enablement/disablement?**
    * Planned for Alpha release

### Rollout, Upgrade and Rollback Planning

*This section must be completed when targeting beta graduation to a release.*

* **How can a rollout or rollback fail? Can it impact already running workloads?**
* **What specific metrics should inform a rollback?**
* **Were upgrade and rollback tested? Was the upgrade→downgrade→upgrade path tested?**
* **Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?**

### Monitoring Requirements

*This section must be completed when targeting beta graduation to a release.*

* **How can an operator determine if the feature is in use by workloads?**
    * Disable the default taint-manager
    * Create custom taint-based pod eviction
* **How can someone using this feature know that it is working for their instance?**
    * Node taints and taint-based pod eviction work as usual.
* **What are the reasonable SLOs (Service Level Objectives) for the enhancement?**
    * Taint-based pod eviction latency
* **What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?**
    * Taint-based pod eviction latency 
* **Are there any missing metrics that would be useful to have to improve observability of this feature?**
    * Taint-based pod eviction latency 

### Dependencies

* **Does this feature depend on any specific services running in the cluster?** 
    * N/A

### Scalability

* **Will enabling / using this feature result in any new API calls?**
    * No
* **Will enabling / using this feature result in introducing new API types?** 
    * No
* **Will enabling / using this feature result in any new calls to the cloud provider?**
    * No
* **Will enabling / using this feature result in increasing size or count of the existing API objects?**
    * No
* **Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?**
    * No
* **Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?**
    * No

### Troubleshooting

* **How does this feature react if the API server and/or etcd is unavailable?**
    * The same behavior as the current version: fail to add taint and/or evict pods on tainted nodes 
* **What are other known failure modes?**
    * No
* **What steps should be taken if SLOs are not being met to determine the problem?**
    * If the default or custom taint-manager is working or not. 

## Implementation History

* 2023-03-06: Initial KEP published. 

## Drawbacks

## Alternatives

Keeping the code as-is is an option that will stop cluster operators from coming up with custom `TaintManager`s however it comes with the increased maintenance burden on the kubernetes developers and also makes any future extensions hard.


## Infrastructure Needed (Optional)

N/A

## Note
`taint-manager`, `TaintManager` and `NoExecuteTaintManager` are used interchangeably in this doc.
