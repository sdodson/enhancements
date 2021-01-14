---
title: eus-upgrades-mvp
authors:
  - "@sdodson"
reviewers:
  - @darkmuggle
  - @soltysh
  - @sttts
  - @deads2k
  - @ecordell
approvers:
  - "@eparis"
  - "@derekwaynecarr"
  - "@crawford"
  - "@pweil-"
creation-date: 2020-01-12
last-updated: 2020-01-12
status: provisional
see-also:
replaces:
superseded-by:
---

# EUS Upgrades MVP

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Operational readiness criteria is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

This enhancement outlines a set of cross platform improvements meant to ensure
the safety of multiple back-to-back minor version upgrades associated with EUS
to EUS upgrades. Additionally, we outline improvements applicable to all
upgrades meant to reduce the duration of upgrades and workload disruption.

All of the work detailed herein is pre-requisite for any future attempts to skip
upgrade steps such as reboots between specific minor releases.

## Motivation

The introduction of EUS creates a subset of clusters which we expect will run
4.6 for a year or more then upgrade rapidly, though serially, from 4.6 to 4.10.
This rapid upgrade introduces the risk that those clusters may upgrade faster
than is safe due constraints imposed by OpenShift, the upstream components of
OpenShift, or deployed workloads.

This also creates a scenario where admins wish to reduce both the duration and
the disruption to workload associated with the upgrade.

### Goals

- We will inform admins of incompatibilities between their current usage and the
final upgrade target
- We will provide guidance on expected upgrade duration scaled by node count exclusive
of the variability associated with workload rescheduling (ie: everything except
Worker MCP rollout)
- We will inhibit upgrades to the next minor version whenever necessary to preserve
cluster stability and supportability, ie:
  - Respecting version skew policies between OS managed components (kubelet, crio, RHCOS)
  and operator managed components (Kubernetes API, OpenShift API, etc)
  - Respecting version skew policies of OLM managed Operators
- We will ensure that CVO managed operators upgrade as quickly as is safe to do so
- We will reduce workload disruption associated with pod rescheduling during rolling
reboots


### Non-Goals

- This enhancement does not attempt to remove *any* steps along the serial
  upgrade path from 4.6 to 4.7 to 4.8 to 4.9 to 4.10 *including* reboots. A
  follow-up enhancement will address those concerns.

## Proposal

This is where we get down to the nitty gritty of what the proposal actually is.

### User Stories

Detail the things that people will be able to do if this is implemented. Include
as much detail as possible so that people can understand the "how" of the
system. The goal here is to make this feel real for users without getting bogged
down.

Include a story on how this proposal will be operationalized:  lifecycled,
monitored and remediated at scale.

#### As an admin preparing for 4.6 to 4.10 EUS upgrade I'm alerted to API removals

We have pre-existing mechanisms to alert admins to their use of APIs which are
slated for removal in a future release. Currently those mechanisms do not look
forward far enough to span 4.6 to 4.10, we will need to add the ability to
backport that API metadata to 4.6 EUS versions once we have a firm understanding
of all removals between 4.6 and 4.10.

#### As an admin I cannot violate OpenShift's defined Kubelet to API version skew

The MCO, or another suitable component, must set Upgradeable=False in order to
prevent upgrades whenever a managed node would violate Kubelet to API version
skew defined by OpenShift. For instance, assume OCP 4.6, 4.7, and 4.8 map to
Kubernetes versions 1.19, 1.20, 1.21. If OpenShift chooses to support only
version skew of N-1 the MCO would inhibit upgrades to 4.8 until all managed
nodes had upgraded to 4.7.

Note, OpenShift may choose to define different version skew constraints than
upstream Kubernetes. We may also change these in the future when our internal
testing indicates it is safe to do so, therefore this should be implemented in a
manner which allows us to change the constraints in a future update to the MCO.

These changes will need to be backported to 4.7 prior to 4.10 GA.

#### As the maintainer of a Market Place Operator I may define either maxKubeVersion or maxOCPVersion

We should allow maintainers to define an inclusive version range for their Operator.

#### As an admin I cannot violate defined OLM managed Operator compatibility

If a managed Operator defines a maxKubeVersion or maxOCPVersion which would be
violated by upgrading to the next minor OLM must set Upgradeable=False and
enumerate the constraint in the condition message.

Note, this is up for debate as to what the behavior should be in absence of a
defined maximum version.

#### As an admin I have estimated upgrade duration data scaled by node count

With support of the PerfScale team the Over The Air team will measure and
analyze upgrade timings using simulated workloads at a defined capacity and
load. This analysis will be used to identify optimizations and set expectations
on a per minor version basis as to the duration of the upgrade.

Due to the variability in workload deployment these measurements will be
*exclusive of Worker MCP rollout*. The simulated workload is intended only to
ensure that our measurements more closely mimic real world usage, not upgrades
of clusters with zero workload.

#### As an admin, upgrades of CVO managed components are as fast as safe

In 4.6 and 4.7 we identified a number of DaemonSets which were not critical to
workload availability which could rollout in a more parallel manner. There are a
few remaining DaemonSets which could be made more parallel after confirming a
canary deployment passes such as multus, sdn, ovs.

Once these new deployment patterns have been vetted in 4.8 or 4.9 these changes
should be backported through to 4.6.

#### As an admin, my workloads within a failure domain are rescheduled a minimum number of times

Currently the MCO cycles nodes in a deterministic but unintentional manner. The
workload on that host which can be relocated scatters to other hosts according
to scheduling rules. This has been measured and shown that approximately 70% of
all reschedulable pods are rescheduled more than necessary if they were
rescheduled in an optimal manner. Optimal rescheduling should be closer to (1 /
node_count_per_failure_domain) percentage of pods being rescheduled more than
once during a rolling restart of a given failure domain.

We should make the MCO topology aware so that it upgrades all of the host in one
failure domain because anti-affinity scheduling is used in most scenarios. We
should provide some mechanism to rank upgraded nodes as slightly higher priority
in scheduler configuration. Perhaps a timestamp of configuration generation that
the scheduler then biases toward nodes with the greatest timestamps as a last in
chain priority.

We should consider enabling the rescheduler as well to ensure that when the
upgrade completes we rebalance so that the last node ends up with proportionate
workload.

### Implementation Details/Notes/Constraints [optional]

The main caveat of what's outlined here is that it does not reduce reboots
which, for many reasons, is highly desired by some admins. The rationale for not
addressing that in this enhancement is that this enhancement targets
improvements which benefit all upgrades.

The epic / user story to backport API removal notifications to 4.6 assumes that
we will require anyone upgrading along an EUS specific upgrade path to update to
a 4.6 version which is specific to EUS. This is necessary because only those who
follow the EUS upgrade path have 4.10 as their next logical target version. We
should not alert non EUS clusters of removals in 4.10 until they've upgraded to
a later minor version.

### Risks and Mitigations

The improvements here do not seem to introduce risk unto themselves. However
failure to implement some of these such as the version skew constraints would
impose significant risk to supportability in clusters which may upgrade more
rapidly than is safe to do so.

Additionally, the timeframe for final delivery extends three releases into the
future. This allows for design iteration during the 4.8 release cycle with
implementation focused on the 4.9 and 4.10 release cycles.

## Design Details

### Open Questions [optional]

As this is a bit of a meta-enhancement calling out epic level work it should be
no surprise that there will be implementation details unaccounted for today. We
will need to decide on an epic by epic level which require enhancements of their
own.

While I believe all of this work to be prerequisite to taking on work to skip
upgrade steps that may not be widely agreed upon.

### Test Plan

Each epic should prescribe their own test plans.

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

#### Examples

These are generalized examples to consider, in addition to the aforementioned [maturity levels][maturity-levels].

##### Dev Preview -> Tech Preview

- Ability to utilize the enhancement end to end
- End user documentation, relative API stability
- Sufficient test coverage
- Gather feedback from users rather than just developers
- Enumerate service level indicators (SLIs), expose SLIs as metrics
- Write symptoms-based alerts for the component(s)

##### Tech Preview -> GA

- More testing (upgrade, downgrade, scale)
- Sufficient time for feedback
- Available by default
- Backhaul SLI telemetry
- Document SLOs for the component
- Conduct load testing

**For non-optional features moving to GA, the graduation criteria must include
end to end tests.**

##### Removing a deprecated feature

- Announce deprecation and support policy of the existing feature
- Deprecate the feature

### Upgrade / Downgrade Strategy

Despite the topic area, this work does not actually change Upgrade or Downgrade
strategy.

### Version Skew Strategy

N/A

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.

## Drawbacks

The idea is to find the best form of an argument why this enhancement should _not_ be implemented.

## Alternatives

Rather than having MCO enforce version skew policies between OS managed
components and operator managed components it could simply set Upgradeable=False
whenever a rollout is in progress. This would preclude minor version upgrades in
situations where a z-stream rollout stalls for some reason or another. We also
know of some customers who have maintenance windows such that they may be in a
perpetual state of MCP rollout.

We could also consider simply waiting for all MachineConfigPool rollouts to have
completed before the MCO considers itself upgraded. This has the downside of
shifting highly variable workload dependent operation into the scope of the
upgrade which is something we track very closely both in terms of success and
duration.


## Infrastructure Needed [optional]

N/A
