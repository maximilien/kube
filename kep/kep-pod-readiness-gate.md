---
title: KEP Template
authors:
  - "@nimak"
  - "@maximilien"
owning-sig: sig-xxx
participating-sigs:
  - sig-apps
  - sig-network
reviewers:
  - TBD
approvers:
  - TBD
editor: TBD
creation-date: 2020-01-18
last-updated: 2020-01-18
status: implementable
see-also:
  - "/keps/sig-aaa/20190101-we-heard-you-like-keps.md"
---

# PodReadinessGate to support conditions defaulting to true

## Table of Contents

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories [optional]](#user-stories-optional)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
  - [Implementation Details/Notes/Constraints [optional]](#implementation-detailsnotesconstraints-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
    - [Examples](#examples)
      - [Alpha -&gt; Beta Graduation](#alpha---beta-graduation)
      - [Beta -&gt; GA Graduation](#beta---ga-graduation)
      - [Removing a deprecated flag](#removing-a-deprecated-flag)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Implementation History](#implementation-history)
- [Drawbacks [optional]](#drawbacks-optional)
- [Alternatives [optional]](#alternatives-optional)
- [Infrastructure Needed [optional]](#infrastructure-needed-optional)
<!-- /toc -->

**NOTE**: all items in [square braket] are meta comments from the [kep-template](kep/kep-template.md)

## Release Signoff Checklist

[**ACTION REQUIRED:** In order to merge code into a release, there must be an issue in [kubernetes/enhancements] referencing this KEP and targeting a release milestone **before [Enhancement Freeze](https://github.com/kubernetes/sig-release/tree/master/releases)
of the targeted release**.]

[For enhancements that make changes to code or processes/procedures in core Kubernetes i.e., [kubernetes/kubernetes], we require the following Release Signoff checklist to be completed.]

[Check these off as they are completed for the Release Team to track. These checklist items _must_ be updated for the enhancement to be released.]

- [ ] kubernetes/enhancements issue in release milestone, which links to KEP (this should be a link to the KEP location in kubernetes/enhancements, not the initial KEP PR)
- [ ] KEP approvers have set the KEP status to `implementable`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [ ] Graduation criteria is in place
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[**Note:** Any PRs to move a KEP to `implementable` or significant changes once it is marked `implementable` should be approved by each of the KEP approvers. If any of those approvers is no longer appropriate than changes to that list should be approved by the remaining approvers and/or the owning SIG (or SIG-arch for cross cutting KEPs).]

[**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.]

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://github.com/kubernetes/enhancements/issues
[kubernetes/kubernetes]: https://github.com/kubernetes/kubernetes
[kubernetes/website]: https://github.com/kubernetes/website

## Summary

We would like to introduce a new label that, once present on a pod, works as a way to indicate to the ReplicaSet controller that under exactly similar conditions, pods with the label should be preferred for removal over pods without the label.

## Motivation

Currently, the ReplicaSet controller [has a defined set of rules](https://github.com/kubernetes/kubernetes/blob/6a19261e96b12c83e6c69d15e2dcca8089432838/pkg/controller/controller_utils.go#L831-L873) on how to decide on which pods to remove when scaling down.

Particulalry, if it needs to choose between pods that are in a `Running` state, the order in which it decides which pods to remove looks like the following

```go
	// TODO: take availability into account when we push minReadySeconds information from deployment into pods,
	//       see https://github.com/kubernetes/kubernetes/issues/22065
	// 5. Been ready for empty time < less time < more time
	// If both pods are ready, the latest ready one is smaller
	if podutil.IsPodReady(s.Pods[i]) && podutil.IsPodReady(s.Pods[j]) {
		readyTime1 := podReadyTime(s.Pods[i])
		readyTime2 := podReadyTime(s.Pods[j])
		if !readyTime1.Equal(readyTime2) {
			return afterOrZero(readyTime1, readyTime2)
		}
	}
	// 6. Pods with containers with higher restart counts < lower restart counts
	if maxContainerRestarts(s.Pods[i]) != maxContainerRestarts(s.Pods[j]) {
		return maxContainerRestarts(s.Pods[i]) > maxContainerRestarts(s.Pods[j])
	}
	// 7. Empty creation time pods < newer pods < older pods
	if !s.Pods[i].CreationTimestamp.Equal(&s.Pods[j].CreationTimestamp) {
		return afterOrZero(&s.Pods[i].CreationTimestamp, &s.Pods[j].CreationTimestamp)
	}
```
To summarize, when all pods are `Running` it prefers them for removal in the order below:

```
  - pods with less ready time over those with more ready time
  - pods with higher restart counts over those with lower restart counts
  - pods with empty creation time over newer pods over older pods
```

However, this leaves no room for higher level controllers to influence or enforce removal of particular pods when they have knoweledge potentially missing to the lower level controllers (e.g. the `ReplicaSet`). In this proposal, we would like to suggest a mechanism for the higher level controls to be able to intervene in the process of pod removal. 

### Goals

- give control over pod lifecycle to higher level k8s controllers

### Non-Goals

- have the higher level controllers directly deal with pod removal
- expose any extra information about pod lifecyle management to higher level controllers.

## Proposal

We would like to propose a modification to the pod ranking algorithm that ranks pods with a given label as higher priority for removal if pods are in a `Running` state. We propose this label to be named `kubernetes.io/prefer-for-scale-down`, and checks for pod removal when they are all in the `Running` state to have another check like the one added to the list below:


```diff
+ - pods with the label kubernetes.io/prefer-for-scale-down over the others
  - pods with less ready time over those with more ready time
  - pods with higher restart counts over those with lower restart counts
  - pods with empty creation time over newer pods over older pods
```

### User Stories

#### Story 1

At a high level the workflow is something like the following:

1. Knative creates a deployment with custom PodReadinessGate set to true for its Pods.
2. The deployment creates X number of Pods as specific in deployment's replicaCount
3. At the time of scaledown:
 - Knative autoscaler has knowledge about which pods to remove
 - Knative autoscaler patches the pods it prefers for removal with the label `kubernetes.io/prefer-for-scale-down`
 - Kubernetes notices change in pod count expectations and removes the unready Pods following this comparison.

### Implementation Details/Notes/Constraints [optional]

ReplicaSet controller has some built-in logic for which Pods to delete first. Allowing users to specify this requires some API & controller changes. The modification would just add another check when ranking the pods, from the code snippet above.

### Risks and Mitigations

We haven't identified any particular risk for the proposal. The ability for patching pods with a particular label is already managed via the security policies the k8s defines and only those with the ability to patch a pod can apply the label. 

The label itself does not make for any removal decision and only helps the `ReplicaSet`controller with prioritization at the time of scaling down.

### Test Plan

[**Note:** *Section not required until targeted at a release.*]

[Consider the following in developing a test plan for this enhancement:]
[- Will there be e2e and integration tests, in addition to unit tests?]
[- How will it be tested in isolation vs with other components?]

[No need to outline all of the test cases, just the general strategy.]
[Anything that would count as tricky in the implementation and anything particularly challenging to test should be called out.]

[All code is expected to have adequate tests (eventually with coverage expectations).]
[Please adhere to the [Kubernetes testing guidelines][testing-guidelines] when drafting this test plan.]

[testing-guidelines]: https://git.k8s.io/community/contributors/devel/sig-testing/testing.md

### Graduation Criteria

[**Note:** *Section not required until targeted at a release.*]

[Define graduation milestones.]

[These may be defined in terms of API maturity, or as something else. Initial KEP should keep]
[this high-level with a focus on what signals will be looked at to determine graduation.]

[Consider the following in developing the graduation criteria for this enhancement:]
[- [Maturity levels (`alpha`, `beta`, `stable`)][maturity-levels]]
[- [Deprecation policy][deprecation-policy]]

[Clearly define what graduation means by either linking to the [API doc definition](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning),]
[or by redefining what graduation means.]

[In general, we try to use the same stages (alpha, beta, GA), regardless how the functionality is accessed.]

[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/

#### Examples

[These are generalized examples to consider, in addition to the aforementioned [maturity levels][maturity-levels].]

##### Alpha -> Beta Graduation

[- Gather feedback from developers and surveys]
[- Complete features A, B, C]
[- Tests are in Testgrid and linked in KEP]

##### Beta -> GA Graduation

[- N examples of real world usage]
[- N installs]
[- More rigorous forms of testing e.g., downgrade tests and scalability tests]
[- Allowing time for feedback]

[**Note:** Generally we also wait at least 2 releases between beta and GA/stable, since there's no opportunity for user feedback, or even bug reports, in back-to-back releases.]

##### Removing a deprecated flag

[- Announce deprecation and support policy of the existing flag]
[- Two versions passed since introducing the functionality which deprecates the flag (to address version skew)]
[- Address feedback on usage/changed behavior, provided on GitHub issues]
[- Deprecate the flag]

[**For non-optional features moving to GA, the graduation criteria must include [conformance tests].**]

[conformance tests]: https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md

### Upgrade / Downgrade Strategy

[If applicable, how will the component be upgraded and downgraded? Make sure this is in the test plan.]

[Consider the following in developing an upgrade/downgrade strategy for this enhancement:]
[- What changes (in invocations, configurations, API use, etc.) is an existing cluster required to make on upgrade in order to keep previous behavior?]
[- What changes (in invocations, configurations, API use, etc.) is an existing cluster required to make on upgrade in order to make use of the enhancement?]

### Version Skew Strategy

[If applicable, how will the component handle version skew with other components? What are the guarantees? Make sure
this is in the test plan.]

[Consider the following in developing a version skew strategy for this enhancement:]
[- Does this enhancement involve coordinating behavior in the control plane and in the kubelet? How does an n-2 kubelet without this feature available behave when this feature is used?]
[- Will any other components on the node change? For example, changes to CSI, CRI or CNI may require updating that component before the kubelet.]

## Implementation History

[Major milestones in the life cycle of a KEP should be tracked in `Implementation History`.]
[Major milestones might include]

[- the `Summary` and `Motivation` sections being merged signaling SIG acceptance]
[- the `Proposal` section being merged signaling agreement on a proposed design]
[- the date implementation started]
[- the first Kubernetes release where an initial version of the KEP was available]
[- the version of Kubernetes where the KEP graduated to general availability]
[- when the KEP was retired or superseded]

## Drawbacks [optional]

[Why should this KEP _not_ be implemented.]

## Alternatives [optional]

[Similar to the `Drawbacks` section the `Alternatives` section is used to highlight and record other possible approaches to delivering the value proposed by a KEP.]

## Infrastructure Needed [optional]

[Use this section if you need things from the project/SIG.]
[Examples include a new subproject, repos requested, github details.]
[Listing these here allows a SIG to get the process for these resources started right away.]
