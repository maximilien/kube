---
title: KEP Pod Readiness Gate
authors:
  - "@nimak"
  - "@maximilien"
  - "@duglin"
owning-sig: sig-apps
participating-sigs:
  - sig-apps
  - sig-network
reviewers:
  - TBD
approvers:
  - TBD
editor: TBD
creation-date: 2020-02-04
last-updated: 2020-01-31
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
<!-- /toc -->

**NOTE**: all items in [square braket] are meta comments from the [kep-template](kep/kep-template.md)

## Release Signoff Checklist

**ACTION REQUIRED:** In order to merge code into a release, there must be an issue in [kubernetes/enhancements] referencing this KEP and targeting a release milestone **before [Enhancement Freeze](https://github.com/kubernetes/sig-release/tree/master/releases)
of the targeted release**.

Check these off as they are completed for the Release Team to track. These checklist items _must_ be updated for the enhancement to be released.

- [X] kubernetes/enhancements issue in release milestone, which links to KEP (this should be a link to the KEP location in kubernetes/enhancements, not the initial KEP PR)
- [ ] KEP approvers have set the KEP status to `implementable`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [ ] Graduation criteria is in place
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

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
