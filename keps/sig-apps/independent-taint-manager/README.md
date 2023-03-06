# KEP-3902:  Decouple Taint-based Pod Eviction from Node LifeCycle Controller 
<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Why the use of  tolerations for <code>NoExecute</code> taints is not good enough for custom pod eviction?](#why-the-use-of--tolerations-for--taints-is-not-good-enough-for-custom-pod-eviction)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories (Optional)](#user-stories-optional)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
    - [Story 4](#story-4)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Current implementation](#current-implementation)
  - [Proposed Controllers](#proposed-controllers)
  - [Custom taint-manager](#custom-taint-manager)
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

In Kubernetes, `NodeLifeCycleController` applies predefined `NoExecute` taints (e.g., `Unreachable`, `NotReady`) when the nodes are determined to be unhealthy. After the nodes get tainted, the `taint-manager` does its due diligence to start deleting running pods on those nodes based on `NoExecute` taints, which can be added by anyone.

In this KEP, we propose to 

* Decouple `taint-manager` that performs taint-based pod eviction from `NodeLifeCycleController` and make them two separate controllers: `NodeLifeCycleManager` to add taints to unhealthy nodes and `TaintManager` to perform pod deletion on nodes tainted with NoExecute effect.

* Allow users to opt-out from the taint-based eviction while still retaining the overall functionality provided by `NodeLifeCycleController.`

## Motivation

* Whether or not evict a stateful pod with local storage on `NoExecute` can depend on the actual taint conditions, workload status and other conditions. The decision may require information from the external systems and controllers, which have more knowledge and context about the workloads.  A custom `taint-manager`  will be able to control how and when a pod is evicted or deleted, including making calls to external systems and take actions on  `NoExecute` taints with other conditions such as unhealthy network. 
* Starting kubernetes 1.27, with the removal of the flag `enable-taint-manager`, replacing the default `NoExecute` taint-manager with a custom one would be more challenging. `NoExecute` taints can disable the no-execute taint manager, but such disabling can only be achieved through a mutating webhook on all pods, and it can interact negatively with the upstream `DefaultTolerations` webhook.
* The requirement of custom taint-based eviction is not unique. It exists already in Kubernetes, where we call it `FullyDistributionMode`. When all nodes are unhealthy, `NodeLifeCycleController` chooses not to apply `NoExecute` taints properly. In a real-world case, the integrator may choose to define the disruption rate/budget, etc.
* `NodeCycleController` combines two independent functions: adding a pre-defined set of `NoExecute` taints to unhealthy nodes and performing pod eviction on arbitrary `NoExecute` taints. While  `NodeLifecycle` controller responsibility is primarily on Node unhealthy-zone, `TaintManger` is concerned with actioning the taints added by `NodeLifecycle` controller by performing Pods eviction. De-coupling `NoExecuteTaintManager` from `NodeLifecycle` controller in `kube-controller-manager` would allow to implement a single eviction controller to take care of general Pod eviction and deletion for different cases based on workload needs, including `NoExecute` taint-based pod eviction other than node `Unhealthy` and `Unreachable`.   

### Why the use of  tolerations for `NoExecute` taints is not good enough for custom pod eviction? 

Toleration cannot provide a flexible and general enough extension mechanism to handle different Pod eviction and deletion cases to meet various workload requirements. 

* The toleration mechanism is not flexible enough to handle complex stateful applications and workloads that require specific coordination of pod eviction, e.g. checkpointing for certain classes of machine-learning workloads to guarantee data integrity before deletion.
* We operate clusters where we want to disable `NoExecute` eviction wholesale. Leveraging toleration would require adding it to every single pod in a cluster. Specifically, custom tolerations will require to either inject tolerations to all pods via mutating webhooks and/or changing the manifests of all the running workloads. 
    * Custom mutating webhook may interfere with the the[admission plugin](https://github.com/kubernetes/kubernetes/blob/master/plugin/pkg/admission/defaulttolerationseconds/admission.go) that applies the default tolerations if they are trying to apply tolerations for similar taints. 
    * Toleration update in Webhooks can cause conflicts with older version of pods especially during upgrades. 
    * In order to ensure toleration updates are valid, a RBAC mechanism will be required in place. 
* Toleration won’t allow users to leverage the existing tolerations API to interface with the component in charge of handling their termination.
* It’s hard for toleration to interact with other controllers and workload management systems, which have more knowledge and context about workload disruption. It also limits to kubernetes clusters and cannot handle workloads that are split across Kubernetes/Non-Kubernetes platforms.

### Goals

* Move taint-based eviction implementation out of `NodeLifeCycleController` into a separate and independent taint-manager.
* Using the existing `kube-controller-manager ` `--controllers`  flag to enable and disable the default `taint-manager` to support customizable taint-manager controller. 

### Non-Goals

* Any change to the behaviour of the current taint-manager

## Proposal

### User Stories (Optional)

#### Story 1

An operator of stateful workloads that use local persistent volume can disable the default taint-manager and use a custom controller instead. When a node is tainted with `NoExecute` , the custom controller won’t delete the running stateful pods on the tainted node until the data is safe,  e.g., after the application manager successfully checkpoints the data in or migrates the data to a safe place. 

#### Story 2

A cluster orchestrator can design and implement a centralized pod eviction manager that handles all pod eviction and deletions on different `NoExecute`  taints (`Unreachable`, `NotReady` and others) and any custom taints, e.g., acting on the condictions defined in `descheduler`, such as node overloading, etc.

**Story 3**

A custom taint manager can make calls to other controllers or external systems to make a better decision whether, when and how to delete running pods on the tainted nodes (e.g. to prevent quorum loss and/or support different mechanism than Pod Disruption Budget to calculate applications availability).

#### Story 4

A pod may be stuck in terminating state for various reason, e.g.,  `kubelet` is not up and `AppServer` cannot reach `kubelet`, etc. The hanging deletion may prevent new pods starting because the persistent volume claim is not released. A custom taint-manager can perform a forceful deletion as needed.

### Notes/Constraints/Caveats (Optional)

### Risks and Mitigations

## Design Details

### Current implementation

In this current version, `NoExecuteTaintManager` is part of `NodeLifycleController`. It’s created and run if `enable-taint-manager` flag is true.  The flag is deprecated and will be removed in kubernetes 1.27. 

<p align="center"><image src="current-nodelifecycle-controller.png" width=50%></p>

### Proposed Controllers

The proposed design refactors `NodeLifecycleController` into two independent controllers, which are managed by `kube-controller-manager`.

1. `NodeLifecycleController` monitors node health and adds `NotReady` and `Unreachable` taints to nodes 
2. `NoExecuteTaintManager` watches for node and pod updates,  and performs `NoExecute` taint based pod eviction

The existing  kube-controller-manager flag `--controllers` will be used to optionally disable the `NoExecuteTaintManager`.

<p align="center"><image src="new-controllers.png" width=60%></p>

### Custom taint-manager

Opting out from the default `TaintManger`, Kubernetes operators will be able to implement custom logic to handle `NoExecute` taint such as delegating the authority for the termination for some classes of workloads providing high-availability protection guarantees. They can also scale out the usage of `NoExcute` to more use-cases than `NotReady` nodes (e.g. unhealthy network, etc.).

<p align="center"><image src="custom-taint-manager.png" width=90%></p>

### Implementation

A new `NodeLifeCycleManager` is implemented by removing taint-manager related code from `controller/nodelifecycle/node_lifecycle_controller.go`.

A new `NoExecuteTaintManage`r is created a first-class controller managed by` kube-controller-manager`. It’s implemention is based on the current taint-manager  `controller/nodelifecycle/taint-manager.go`.  A new flag is added 

`// NoExecuteTaintManagerConfiguration contains elements describing NoExecuteTaintManagerConfiguration`
`type NoExecuteTaintManagerConfiguration struct {`
`// enables the default NoExecuteTaintManager. MUST be synced with the`
`// corresponding flag of the kube-apiserver.`
`EnableNoExecuteTaintManager bool`
`  ...`
`}`

The creation and startup of the default taint-manager is performed by `kube-controller-manager` . 

`func startNoExecuteTaintManager(ctx context.Context, controllerContext ControllerContext) (controller.Interface, bool, error) {`
`    if !controllerContext.ComponentConfig.NoExecuteTaintManager.EnableNoExecuteTaintManager {
        return nil, false, nil
    }`

```
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

* Unit and integration tests
    * Add nodes taints
    * Act on `NoExecute` to delete running pods
    * Respect taint tolerations 
* Verify passing the existing E2E and conformance tests.
    * Taint a node 
    * Evict the correct set of running pods

### Graduation Criteria

#### Alpha

* Support `kube-controller-manaager` flag `controllers=-taint-manager` to disable the default taint-manager.
* Unit and integration tests completed and passed.

#### Beta

* Performance and scalability tests to verify there is non-negligible performance degradation in node taint-based eviction.
* Update documents to reflect the changes.

#### GA

* Fix all reported bugs if any.

### Upgrade / Downgrade Strategy

We’ll need to upgrade `kube-controller-manager` and` node-lifecycle-controller` and add a new controller `taint-manager`. Accordingly, a downgrade will revert to the previous version of `kube-controller-manager` and `node-lifeycle-controller` and remove the new added `taint-manager` . There should be no impact on the other components during upgrades and downgrades since user-exposed behavior should stay the same except a new flag to enable/disable the default `taint-manager`. This change should be clearly communicated to users.

### Version Skew Strategy

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

*This section must be completed when targeting alpha to a release.*

* **How can this feature be enabled / disabled in a live cluster?**
    * Enabled:  upgrade `kube-controller-manager` and` node-lifecycle-controller` and add a new `taint-manager` controller.
    * Disabled: downgrade will revert to the previous version of `kube-controller-manager` and `node-lifeycle-controller` and remove the new added `taint-manager` . 
* **Does enabling the feature change any default behavior?**
    * No, the default node-life-cycle controller and taint-manager behavior will stay the same.
* **Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?**
    * Yes, we will need to rack back `kube-controller-manager` and `node-lifecycle-controller`. 
* **What happens if we reenable the feature if it was previously rolled back?**
    * Upgrade `kube-controller-manager` and `node-lifycle-controller` and add the new `taint-manager`.
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

While operators can opt-out from taint-based eviction by injecting tolerations for `NoExecute` taints, this would have few major disadvantages over custom implementations of `TaintManager`.

* It won’t allow users to leverage the existing tolerations API to interface with the component in charge of handling their termination.
* It will require to either inject tolerations to all pods via mutating webhooks and/or changing the manifests of all the running workloads.

## Infrastructure Needed (Optional)

N/A
