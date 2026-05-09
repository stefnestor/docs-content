---
navigation_title: "Troubleshoot upgrades"
description: "Common upgrade issues and resolutions."
type: troubleshooting
applies_to:
  stack:
products:
  - id: elasticsearch
---

# Troubleshoot upgrades [troubleshooting-upgrades]

Usually, [{{es}} upgrades](/deploy-manage/upgrade/deployment-or-cluster/elasticsearch.md) proceed smoothly due to [planning](/deploy-manage/upgrade/plan-upgrade.md) and [preparation](/deploy-manage/upgrade/prepare-to-upgrade.md) due dilligence.

This guide outlines {{es}} logs which indicate either upgrade blocking issues or fatal node start-up errors.


## 

To monitor which nodes have been upgraded, use the [CAT nodes](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cat-nodes) API:

```console
GET _cat/nodes?v=true&h=name,ip,version,uptime&s=uptime
```


## Rolling upgrades considerations [upgrade-issues]

{{es}} supports running two versions during a rolling upgrade, from an earlier version to later version. It does not ever support running more than two versions. It does not support two versions beyond the duration of the rolling upgrade.

During a rolling upgrade, the cluster continues to operate normally. New functionality is either inactive or operates in a backward-compatible mode until the last node of earlier version leaves the cluster. New functionality becomes operational when all nodes in the cluster are running the later version. 

Usually, the earlier version nodes only leave the cluster when you shut them down to upgrade them. In this case, the last earlier version node leaves the cluster when there are no more nodes to upgrade. However, it is possible that an earlier version node might temporarily or permanently (until intervened) leave the cluster before you purposely shut it down due to [cluster fault detection](/deploy-manage/distributed-architecture/discovery-cluster-formation/cluster-fault-detection.md).

If all the remaining earlier version nodes unexpectedly leave the cluster during an upgrade, the cluster will consider itself to be fully-upgraded, automatically activate new functionality, and leave its backward-compatible mode.

Once that has happened, there is no way to return the cluster to a state that is compatible with the earlier version nodes. Nodes running the earlier version will not be able to join this fully-upgraded cluster. Their {{es}} logs will report `failed to join` issues due to `Caused by` errors like

* `node version [x.x.x] may not join a cluster comprising only nodes of version [y.y.y] or greater`
* `node with version [x.x.x] may not join a cluster with minimum version [y.y.y]`
* `node with system index mappings versions [y.y.y] may not join a cluster with minimum system index mappings versions [x.x.x]`
* `handshake with [NODE_ID] failed: remote node version [x.x.x] is incompatible with local node version [y.y.y]`

{{es}} maintains the data in the data paths of the older nodes and will recover the cluster to health using this data after the nodes are fully upgraded. Therefore, to bring these nodes back into the cluster, upgrade them.

:::{note} :applies_to: { ece:, ess: }
Usually you can "Reapply" your latest [Deployment activity](/deploy-manage/deploy/elastic-cloud/keep-track-of-deployment-activity.md) {{es}} upgrade to finish upgrading. If the node out of cluster causes [Cluster health](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cluster-health) status of `red`, then plans will be blocked for data safety. If this is the case, kindly [contact us](/troubleshoot/index.md#contact-us) with {{ech}} deployment ID or [{{ece}} diagnostic](/troubleshoot/deployments/cloud-enterprise/run-ece-diagnostics-tool.md) flagged `--deployments` for problematic deployment.
:::

If you stop half or more of the master-eligible nodes all at once during the upgrade, the cluster will [become unavailable](/troubleshoot/elasticsearch/discovery-troubleshooting.md#discovery-no-master) due to insufficient [voting configurations](/deploy-manage/distributed-architecture/discovery-cluster-formation/modules-discovery-voting.md).

You must restart all the stopped master-eligible nodes to allow the cluster to re-form. If the re-formed cluster comprises only upgraded nodes, then the cluster will consider itself to be fully-upgraded, automatically activate new functionality, and leave its backward-compatible mode. In this case, upgrade all other nodes running the old version to enable them to join the re-formed cluster. Upgrade the master-eligible nodes last to make it less likely that this occurs.

In a testing or development environment with only one or two master-eligible nodes, you cannot avoid stopping half or more of the master-eligible nodes, so the cluster will always become unavailable at some point during the upgrade. When you restart the master-eligible nodes after this unavailability, the cluster will re-form with a single upgraded node, which is therefore fully-upgraded and will reject older nodes' attempts to re-join the cluster. Upgrade the master-eligible nodes last to avoid these rejections.


## Symptoms

<!-- REQUIRED

Describe what users observe when the problem occurs. Focus on the symptoms themselves, not their causes. Use bullet points. If applicable, include:

- Error messages
- Log output
- Missing or unexpected behavior
- Timeouts or performance issues
-->

## Diagnosis

<!-- OPTIONAL

Use a Diagnosis section when you need to help users narrow down the problem from an initial symptom before providing the resolution. This is especially useful when:

- The initial symptom requires diagnostic steps to identify the specific cause
- Multiple resolutions depend on diagnostic findings
- The same symptom can have multiple root causes

Use numbered steps or bullet points to guide users through the diagnostic process.
-->

## Resolution

<!-- REQUIRED

Provide clear, actionable steps to resolve the issue.

- Order steps from most common to least common to resolve the issue
- Numbered instructions that begin with imperative verb phrases
- Keep each step focused on a single action
- Use the stepper component
- Avoid diagnostic branching unless the problem cannot be resolved linearly.

For complex scenarios, consider these patterns:

- Multiple resolutions or "combo" resolutions: When multiple solutions can be applied together or independently, present them as a list of options (users can choose one or combine multiple approaches).

- Resolutions that differ by deployment type: When steps differ significantly by deployment type ({{ecloud}} versus self-managed versus ECK), organize by deployment type using clear headings.

- Separating diagnosis from causes and resolution: When multiple resolutions depend on diagnostic findings, use a separate Diagnosis section before Resolution to help users identify their specific situation first.

For more information about the stepper component, refer to [the syntax guide](https://elastic.github.io/docs-builder/syntax/stepper/).
-->

```markdown
:::::{stepper}

::::{step} [Step title]
[Step description or instruction - begin with an imperative verb]
::::

::::{step} [Step title]
[Step description or instruction - begin with an imperative verb]
::::

::::{step} [Step title]
[Step description or instruction - begin with an imperative verb]
::::

:::::
```

## Best practices

<!-- OPTIONAL BUT RECOMMENDED 

Explain how to avoid this issue in the future. Use bullet points. Do not restate general product best practices or guidance that applies broadly beyond this issue.
-->

## Resources

<!-- OPTIONAL 

Link to related documentation for deeper context. These links are supplementary — all information required to fix the issue should already be on this page.

Avoid linking to GitHub issues, pull requests, or internal discussions. Resources should be stable, user-facing documentation.
-->

- [Related documentation link]
- [Contrib/upstream reference]