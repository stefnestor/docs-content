---
navigation_title: "Troubleshoot Upgrade Assitant"
description: "Common upgrade issues and resolutions."
type: troubleshooting
applies_to:
  stack: ga
products:
  - id: kibana
  - id: cloud-enterprise
  - id: cloud-hosted
  - id: cloud-kubernetes
  - id: elasticsearch
---

# Troubleshoot Upgrade Assistant [troubleshooting-upgrade-assistant]

You should run the [Upgrade Assistant](/deploy-manage/upgrade/prepare-to-upgrade/upgrade-assistant.md) from the current major's latest minor and patch before upgrading to the next major stack version. You should end up with a list of steps each filled-in with a green checkmark. Failure to do so can cause your cluster to encounter one of the [common upgrade issues](/troubleshoot/elasticsearch/troubleshooting-upgrades.md).

We have compiled the most common errors and resolutions that clusters encounter while running the **Upgrade Assistant** for your reference.

## Migration errors [troubleshooting-upgrade-assistant-migrations]

{{es}} indices are [compatible for sequential major versions](/deploy-manage/upgrade/deployment-or-cluster/reading-indices-from-older-elasticsearch-versions.md). You must [migrate older index versions](https://www.elastic.co/docs/api/doc/elasticsearch/group/endpoint-migration) to avoid [index compatibility errors](/troubleshoot/elasticsearch/troubleshooting-upgrades.md#troubleshooting-upgrades-errors-bootstrap-index) which would block a node's start-up during its upgrade to a later version.

The **Upgrade Assistant** automatically performs these migrations [on behalf of the user](/deploy-manage/upgrade/prepare-to-upgrade/upgrade-assistant.md#_required_permissions_11). We recommend using the {{kib}} UI automation and only falling back to the {{es}} API as required for error resolution. Critical errors requiring intervention can surface under steps:

* **Migrate system indices**: If the error `System indices migration failed` reports after you clicked "Migrate indices", then click "View Migration Details" to show the output of {{es}}'s [Get feature migration information API]({{es-apis}}operation/operation-migration-get-feature-upgrade-status) for errors.
* **Review deprecated settings and resolve issues** for {{es}}: If the error `Old index with a compatibility version` reports, determine which of the available "Action" items you prefer between reindexing, setting read-only mode, or deleting. If you choose reindexing, any issues encountered will surface under a "Reindexing error" banner.

Reindexing errors commonly occur due to configuration issues with templates or index blocks.

### Index block errors [troubleshooting-upgrade-assistant-migrations-blocks]

{{es}} can surface `cluster_block_exception` [index blocks](elasticsearch:/reference/elasticsearch/index-settings/index-block.md) errors. These blocks are protections for an index and usually purposely configured, so you should review your index's architecture before removing block.

The most common is `FORBIDDEN/8/index write (api)` from an index have `index.blocks.write` setting enabled. The error would appear like:

```json
{
    "cause":
    {
        "reason": "index [.test-reindexed-for-8] blocked by: [FORBIDDEN/8/index write (api)];",
        "type": "cluster_block_exception"
    },
    "id": "XXXXX",
    "index": ".test-reindexed-for-8",
    "status": 403,
    "type": "_doc"
}
```

For example, this can occur when your {{es}} [Reindex API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-reindex)'s target index previously existed and had a write block enabled after ingestion was completed.

It is also possible that a cluster experiencing disk watermark issues might surface `FORBIDDEN/12/index read-only / allow delete (api)`. [Watermark errors](/troubleshoot/elasticsearch/fix-watermark-errors.md) should be resolved before proceeding.

The **Upgrade Assistant** can proactively warn you of possible watermark issues which might exacerbate during upgrade with message:

```text
Get disk usage on all nodes below the value specified in cluster.routing.allocation.disk.watermark.low (nodes impacted: [NODE_NAME, NODE_NAME])
```

This message should be treated as a warning and reviewed to ensure sufficient disk for rolling upgrade before proceeding.

### Template errors [troubleshooting-upgrade-assistant-migrations-blocks]

Reindexing can surface issues which would normally surface during index creation. The most common relate to conflicting [Index Templates](/manage-data/data-store/templates.md) where:

* The index is blocked from creating.

  Conflicting templates can prevent index creation with `IllegalStateException` errors such as:

  * `unable to create new index [X] because it would match composable template [Y]`
  * `unable to create new index [X] because it would match legacy template [Y]`

  You will need to resolve template conflicts before proceeding. You can investigate `overlapping` templates with the [Simulate an index API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-indices-simulate-index-template).

* The index creates but inherits incorrect settings or mappings.

  On historical {{es}} versions, user-created templates could impair [system indices](elasticsearch://reference/elasticsearch/rest-apis/api-conventions.md#system-indices) from creating correctly. The symptom of this would most commonly surface as `.security` reindexing or node start-up errors such as:

  * `mapping set to strict, dynamic introduction of [allow_restricted_indices] within [indices] is not allowed`
  * `missing _meta field in mapping [_doc] of index [.security]`
  * `failed to notify ClusterStateListener`
  * `cannot read security-version string in index .security`
  * `security index is not on the current version - the native realm will not be operational until the upgrade API is run on the security index`

  On user-managed indices, indices inheriting from an unexpected template would most commonly surface during reindexing as errors such as:

  * `mapper_parsing_exception` due to `reason` of `failed to parse field [V] of type [X] in document with id 'Y'. Preview of field's value: Z"`
  * `limit of total fields [X] in index [Y] has been exceeded` due to surpassing [mapping limit setting](elasticsearch://reference/elasticsearch/index-settings/mapping-limit.md)


:::{tip}
Be careful to review any index templates setup to target all indices, for example with `index_patterns` against `*` or `.*`. You should avoid these as they can both cause errors during index creation or unexpected behavior after index creation.
:::
