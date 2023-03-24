<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: runAs-for-supporting-steps
authors:
  - "@SaschaSchwarze0"
reviewers:
  - "@adambkaplan"
  - "@qu1queee"
approvers:
  - "@adambkaplan"
  - "@qu1queee"
creation-date: 2023-03-24
last-updated: 2023-03-24
status: implementable
---

# RunAs user and group for supporting steps

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, DA
- [ ] User-facing documentation is created in [docs](/docs/)

## Summary

This ship proposes a new configuration option that allows to run the supporting steps that Shipwright is adding to a BuildRun (the source steps, the image mutation step) as the same user that runs the build strategy steps.

## Motivation

In build strategies, authors define the steps that are needed to turn source code into a container image. For a complete BuildRun, shipwright prepends, and appends additional steps. Prepended steps take care of the loading of the source code that is defined in the Build. Appended steps process the image that was built.

In its [configuration](https://github.com/shipwright-io/build/blob/v0.11.0/docs/configuration.md), Shipwright allows to define the container templates for these supporting steps. The container template can include a security context with a runAsUser and runAsGroup. This template applies to all BuildRuns. The runAs user and group of the source step has two consequences:

1. The source code on the file system is owned by this user and group.
2. As long as we have creds-init enabled, the `.docker` directory in `/tekton/home` is owned by this user and group.

Build strategy authors can also define a securityContext for each step where they can define a runAsUser and runAsGroup.

Different sample build strategies need to run as different users and groups. For example:

- [Kaniko is running as user 0](https://github.com/shipwright-io/build/blob/v0.11.0/samples/buildstrategy/kaniko/buildstrategy_kaniko_cr.yaml#L15) (root) due to the way is building the container image.
- [BuildKit runs as user 1000 and group 1000](https://github.com/shipwright-io/build/blob/v0.11.0/samples/buildstrategy/buildkit/buildstrategy_buildkit_cr.yaml#L58-L59)
- [Paketo Buildpacks today run as user 1000 and group 1000](https://github.com/shipwright-io/build/blob/v0.11.0/samples/buildstrategy/buildpacks-v3/buildstrategy_buildpacks-v3_cr.yaml#L40-L41). That's only the case as long as we are using the Bionic builder image there. For Jammy, the Paketo community actually updated the stack to user 1001 and group 1000 at build time. We will need to adapt this in our sample strategy when we switch to Jammy there.
- [ko runs as user 1000 and group 1000](https://github.com/shipwright-io/build/blob/v0.11.0/samples/buildstrategy/ko/buildstrategy_ko_cr.yaml#L55-L56)

All sample build strategies that run as non-root currently contain a [prepare step](https://github.com/shipwright-io/build/blob/v0.11.0/samples/buildstrategy/ko/buildstrategy_ko_cr.yaml#L29-L49). This step ensures that `/tekton/home` is owned by the build strategy runAsUser/runAsGroup so that creds-init also works there IF the source step runs as a different user. With today's default configuration, this is a no-op because all source steps by default run as user=1000 and group=1000 which is the same non-root user of all our sample build strategies. It is only there in case somebody uses a different user, or group for the source step. Though, I think we really never tested this and are currently having an issue because we do not `chown` the source directory as well. Basically if the source step is reconfigured as something else, then the runAsUser/runAsGroup of our sample build strategies may not have full access insight the source directory which can lead to failures.

To improve this, I am suggesting to change our logic to run the supporting steps as the user that is used in the build strategy IF all build strategy steps are defined to run as the same user. This means that all steps will run as the same user. The same logic applies to the group.

### Goals

- Run all Shipwright-injected steps as the same user that is used by the build strategy.
- Remove all prepare steps from sample build strategies. This implies that we will have many build strategies that then fully run as non-root user.
- Change the Paketo sample build strategy to Jammy to prove that non-root strategies that run as a non-root user that is not 1000 actually work.

### Non-Goals

- Read the runAsUser and runAsGroup from the step image when they are not defined directly on the step. This can be a future extension, but for now we expect the build strategy author to specify runAsUser and runAsGroup even if they match the image configuration.

### User Stories

#### Story 1

As a Shipwright User, I want to have sample build strategies that really run everything as a non-root user.

#### Story 2

As a Shipwright Build Strategy author, I want to read about recommendations to specify runAsUser and runAsGroup so that I correctly define the steps.

## Proposal

### Base image changes

TBD we may need some changes if the user must be in /etc/passwd

### BuildRun reconciler changes

The BuildRun reconciler is changed to check the steps of the build strategy. If all of the included steps define a runAsUser or runAsGroup that is always the same, then the Shipwright-added steps will be changed to run as this user and/or group.

Otherwise, the runAsUser and runAsGroup that are defined in the configuration's container templates are used.

### Configuration

TBD: do we want this as a configurable feature? Given I want to change the sample build strategies to leverage that capability, it would mean to be on by default. I tend to make it NOT configurable because the only change will be that the source step runs as a different user if the strategy runs as a specified user that is not 1000. This should not break things in my opinion.

### Sample updates

The prepare step of all build strategies is removed. The Paketo build strategy is updated to use the Jammy builder and run as user 1001.

### Documentation updates

The build strategy documentation is updated to explain the consequences of defining, or not defining runAsUser and runAsGroup on the steps.

### Test Plan

Unit and integration tests should cover the part that ensures that everything behaves correctly. Our existing e2e tests should be able to cover the changes to the build strategies.

The e2e tests that depend on a Git secret have to succeed locally as they do not run as part of a build.

## Release Criteria

## Risks and Mitigations

- Risk: we might break existing build strategies that run steps as a non-root non-1000 user and were able to work with source code cloned as 1000 without the prepare step hack. I don't think that this is realistically the case. But, it can't be ruled out completely. The release notes and release blog post should highlight the change and link to the build strategy documentation change.

## Drawbacks

## Alternatives

- Stick with the current approach of having a runAsRoot prepare step that changes ownership.

## Implementation History

I spiked on some of the aspects already [here](https://github.com/SaschaSchwarze0/build/commit/b02e1c4278925441ab07fad0c3cf89b3b31f171d) and [here](https://github.com/SaschaSchwarze0/build/commit/9995661392d693c2f1728e695e588837740703c0). Note that the branches that contain those commits won't survive forever which means that the links might break.
