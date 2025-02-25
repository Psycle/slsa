---
title: "Build Type: GitHub Actions Workflow"
layout: standard
hero_text: |
    A [SLSA Provenance](../../provenance/v1-rc1) `buildType` that describes the
    execution of a GitHub Actions workflow.
---

## Description

```jsonc
"buildType": "https://slsa.dev/github-actions-workflow/v1-rc1"
```

This `buildType` describes the execution of a top-level [GitHub Actions]
workflow that builds a software artifact.

Only the following [event types] are supported:

| Supported event type  | Event description |
| --------------------- | ----------------- |
| [`create`]            | Creation of a git tag or branch. |
| [`deployment`]        | Creation of a deployment. |
| [`release`]           | Creation or update of a GitHub release. |
| [`push`]              | Creation or update of a git tag or branch. |
| [`workflow_dispatch`] | Manual trigger of a workflow. |

[`create`]: https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#create
[`deployment`]: https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#deployment
[`release`]: https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#release
[`push`]: https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#push
[`workflow_dispatch`]: https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#workflow_dispatch

This build type MUST NOT be used for any other event type unless this
specification is first updated. This ensures that the [external parameters] are
fully captured and that the semantics are unambiguous. Other event types are
irrelevant for software builds (such as `issues`) or have complex semantics that
may be error prone or require further analysis (such as `pull_request` or
`repository_dispatch`). To add support for another event type, please open a
[GitHub Issue][SLSA Issues].

Note: Consumers SHOULD reject unrecognized external parameters, so new event
types can be added without a major version bump as long as they do not change
the semantics of existing external parameters.

Note: This build type is **not** meant to describe execution of a subset of a
top-level workflow, such as an action, job, or reusable workflow. Only workflows
have sufficient [isolation] between invocations, whereas actions and jobs do
not. Reusable workflows do have sufficient isolation, but supporting both
top-level and reusable would make the schema too error-prone.

[SLSA Issues]: https://github.com/slsa-framework/slsa/issues
[GitHub Actions]: https://docs.github.com/en/actions
[event types]: https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows
[isolation]: /spec/v1.0-rc1/requirements#isolation-strength

## Build Definition

### External parameters

[External parameters]: #external-parameters

All external parameters are REQUIRED unless empty. At most one of `deployment`,
`inputs`, or `release` can be set.

<table>
<tr><th>Parameter<th>Type<th>Description

<tr id="deployment"><td><code>deployment</code><td>object<td>

The non-default [deployment body parameters] for `deployment` events; unset for
other event types. SHOULD omit parameters that have a default value to make
verification easier. SHOULD be unset if there are no non-default values.

Only includes the parameters that are passed to the workflow. As of API version
2022-11-28, this is: `description`, `environment`, `payload`,
`production_environment`, `transient_environment`.

Can be computed from the [github context] using the corresponding fields from
`github.event.deployment`, filtering out default values (see API docs) and using
`original_environment` for `environment`.

<tr id="inputs"><td><code>inputs</code><td>object<td>

The `inputs` from `workflow_dispatch` events; unset for other event types.
SHOULD omit empty inputs to make verification easier. SHOULD be unset if there
are no non-empty inputs.

Can be computed from the [github context] using `github.event.inputs`.

<tr id="release"><td><code>release</code><td>object<td>

The non-default [release body parameters] for `release` events; unset for
other event types. SHOULD omit parameters that have a default value to make
verification easier. SHOULD be unset if there are no non-default values.

Only includes the parameters are passed to the workflow. As of API version
2022-11-28, this is: `body`, `draft`, `name`, `prerelease`, `target_commitish`.

Can be computed from the [github context] using the corresponding fields from
`github.event.release`, filtering out default values (see API docs).

<tr id="vars"><td><code>vars</code><td>object<td>

The [variables] passed in to the workflow. This SHOULD be unset if there are no
vars.

Can be computed from the [vars context] using `vars`.

<tr id="workflow"><td><code>workflow</code><td>object<td>

The workflow that was run. For most workflows, this commit is the source code to
be built.

<tr id="workflow.ref"><td><code>workflow.ref</code><td>string<td>

A git reference to the commit containing the workflow, as either a git ref
(starting with `refs/`) or commit ID (lowercase hex). This is the value passed
in via the event. Only `deployment` events support commit IDs.

Can be computed from the [github context] using `github.ref || github.sha`.

<tr id="workflow.repository"><td><code>workflow.repository</code><td>string<td>

HTTPS URI of the git repository, with `https://` protocol and without `.git`
suffix.

Can be computed from the [github context] using
`github.server_url + "/" + github.repository`.

<tr id="workflow.path"><td><code>workflow.path</code><td>string<td>

The path to the workflow YAML file within the commit.

Can be computed from the [github context] using
`github.workflow_ref`, removing the prefix `github.repository + "/"` and the
suffix `"@" + github.ref`. Take care to consider that the path and/or ref MAY
contain `@` symbols.

</table>

[deployment body parameters]: https://docs.github.com/en/rest/deployments/deployments?apiVersion=2022-11-28#create-a-deployment--parameters
[github context]: https://docs.github.com/en/actions/learn-github-actions/contexts#github-context
[release body parameters]: https://docs.github.com/en/rest/releases/releases?apiVersion=2022-11-28#create-a-release--parameters
[variables]: https://docs.github.com/en/actions/learn-github-actions/variables
[vars context]: https://docs.github.com/en/actions/learn-github-actions/contexts#vars-context

### System parameters

All system parameters are OPTIONAL.

| Parameter | Type     | Description |
| --------- | -------- | ----------- |
| `github`  | object   | A subset of the [github context] as described below. Only includes parameters that are likely to have an effect on the build and that are not already captured elsewhere. |

The `github` object SHOULD contains the following elements:

| GitHub Context Parameter     | Type   | Description |
| ---------------------------- | ------ | ----------- |
| `github.actor_id`            | string | The numeric ID of the user that triggered the initial workflow run. |
| `github.event_name`          | string | The name of the event that triggered the workflow run. |
| `github.repository_id`       | string | The numeric ID corresponding to `externalParameters.workflow.repository`. |
| `github.repository_owner_id` | string | The numeric ID of the user or organization that owns `externalParameters.workflow.repository`. |
| `github.triggering_actor_id` | string | The numeric ID of the user that triggered the rerun, if different than `actor_id`. |

Numeric IDs are used here to provide stable identifiers across account and
repository renames and to detect when an old name is reused for a new entity.

### Resolved dependencies

The `resolvedDependencies` SHOULD contain an entry identifying the resolved
git commit ID corresponding to `externalParameters.workflow`. The dependency's
`uri` MUST be in [SPDX Download Location] format, i.e.
`"git+" + workflow.uri + "@" + workflow.ref`. See [Example].

The `resolvedDependencies` MAY contain additional artifacts known to be input to
the workflow, such as the specific versions of the virtual environments used.

[SPDX Download Location]: https://spdx.github.io/spdx-spec/v2.3/package-information/#77-package-download-location-field

## Run details

### Metadata

The `invocationId` SHOULD be set to `github.server_url + "/actions/runs/" +
github.run_id + "/attempts/" + github.run_attempt`.

## Example

[Example]: #example

```json
{% include_relative examples/v1-rc1/example.json %}
```

Note: The `builder.id` in the example assumes that the build runs under
[slsa-github-generator](https://github.com/slsa-framework/slsa-github-generator).
If GitHub itself generated the provenance, the `id` would be different.

## Version history

### v1

Initial version
