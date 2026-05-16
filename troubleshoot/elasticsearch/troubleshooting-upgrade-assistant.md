---
navigation_title: "Troubleshoot Upgrade Assitant"
description: "Common upgrade issues and resolutions."
type: troubleshooting
applies_to:
  stack:
products:
  - id: kibana
  - id: cloud-enterprise
  - id: cloud-hosted
  - id: cloud-kubernetes
  - id: elasticsearch
---

# Troubleshoot Upgrade assistant [troubleshooting-upgrade-assistant]

You should run the [Upgrade Assistant](/deploy-manage/upgrade/prepare-to-upgrade/upgrade-assistant.md) from the current major's latest minor and patch before upgrading to the next major stack version. You should end up with a filled-in green checkmark on each step.

## Upgrade asssitant errors

The {{kib}} [Upgrade Assistant](/deploy-manage/upgrade/prepare-to-upgrade/upgrade-assistant.md) second step "Migrate system indices" might error `System indices migration failed`. Click "View Migration Details" to show the output of {{es}}'s [Get feature migration information API]({{es-apis}}operation/operation-migration-get-feature-upgrade-status)

- https://support.elastic.dev/knowledge/view/f3c6ea67
```
IllegalStateException[unable to create new index [X] because it would match composable template [Y]];
```
- https://discuss.elastic.co/t/issues-upgrading-elasticsearch-from-8-to-9-due-to-old-7-x-indices-and-restricted-kibana-indices/379133/11
```
IllegalStateException[unable to create new index [X] because it would match legacy template [Y]];
```
- Avoiding "match all" index templates https://support.elastic.dev/knowledge/view/1bbb4cc9
- https://github.com/elastic/kibana/issues/119652#issuecomment-979205209
```
org.elasticsearch.ElasticsearchException: error occurred while reindexing, bulk failures [{"index":".kibana_1-reindexed-for-8","type":"_doc","id":"space_a:visualization:e1d0f010-9ee7-11e7-8711-e7a007dcef99","cause":{"type":"cluster_block_exception","reason":"index [.kibana_1-reindexed-for-8] blocked by: [FORBIDDEN/8/index write (api)];"},"status":403}
```
https://www.elastic.co/guide/en/elasticsearch/reference/current/fix-watermark-errors.html 
- mapper_parsing_exception from legacy template `.security-*` 
https://github.com/elastic/elasticsearch/issues/86801
```
mapping set to strict, dynamic introduction of [allow_restricted_indices] within [indices] is not allowed
```

https://github.com/elastic/kibana/issues/116209

