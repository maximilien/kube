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

We would like to use the user provided readiness conditions as a way to notify the control plane that a given pod needs to stop serving traffic and its respective endpoints need to be removed.

## Motivation

Currently `PodReadinessGate` requires a two-step update to the Pod for a user provided condition to be set to `true`.

1. The `conditionType` is introduced when creating the pod

2. The Pod `status` is patched to set the condition to `true`.

While the behavior works for the initially intended purpose of this field (i.e., more control over when to make the Pod available for routing traffic), reversing this behavior is not easily achieved.

We would like to use the user provided readiness conditions as a way to notify the control plane that a given pod needs to stop serving traffic and its respective endpoints need to be removed.

As such, ideally we would like to create the pod with the custom pod condition set to `true` by default to be able to toggle it to False and retrigger reconciliation at a later point as needed.

`PodReadinessGate` supporting custom conditions defaulting to `true`, appears to be the right approach to it.

### Goals

- `PodReadinessGate` supporting custom conditions defaulting to `true`

[How will we know that this has succeeded?]

### Non-Goals

[What is out of scope for this KEP?]
[Listing non-goals helps to focus discussion and make progress.]

## Proposal

At a high level the workflow is something like the following:

1. Knative creates a deployment with custom PodReadinessGate set to true for its Pods.
2. The deployment creates X number of Pods as specific in deployment's replicaCount
3. At the time of scaledown:
 - Knative selectively toggles the PodReadinessGate condition to false, for the Y number of pods it intends to kill (basically marking them as not ready, based on metrics known to Knative autoscaler)
 - Knative autoscaler patches the replicaCount on the deployment to X - Y
 - Kubernetes notices change in pod count expectations and removes the unready Pods following this comparison.

Essentially the idea is to utilize readiness as a means to make k8s be more decisive about pods to kill when scaling down. This will potentially help with solving similar problems for k8s like discussed here: #45509

We have tested and verified that changing Pod readiness allows for this behavior, but this is not uniformly achievable by failing the ReadinessProbe since configuraion parameters like FailureThreshold and TimeoutSeconds can delay the amount of time it takes for the Pod to become Not Ready.

### User Stories [optional]

[Detail the things that people will be able to do if this KEP is implemented.]
[Include as much detail as possible so that people can understand the "how" of the system.]
[The goal here is to make this feel real for users without getting bogged down.]

#### Story 1

[add here]

### Implementation Details/Notes/Constraints [optional]

ReplicaSet controller has some built-in logic for which Pods to delete first. Allowing users to specify this requires some API & controller changes. 
[add more here]

### Risks and Mitigations

[What are the risks of this proposal and how do we mitigate.]
[Think broadly.]
[For example, consider both security and how this will impact the larger kubernetes ecosystem.]

[How will security be reviewed and by whom?]
[How will UX be reviewed and by whom?]

[Consider including folks that also work outside the SIG or subproject.]

## Design Details

[add here]

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