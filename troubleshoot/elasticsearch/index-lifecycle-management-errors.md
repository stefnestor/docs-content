---
navigation_title: Index lifecycle management errors
mapped_pages:
  - https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-error-handling.html
applies_to:
  stack:
products:
  - id: elasticsearch
---

# Fix {{ilm}} errors [index-lifecycle-error-handling]

[{{ilm}}](../../manage-data/lifecycle/index-lifecycle-management.md) executes actions asynchronously against your cluster's indices based off of conditional-based and time-based logic. These [phase, action, and step executions](/manage-data/lifecycle/index-lifecycle-management/index-lifecycle.md) run sequentially on behalf of the user which last edited the {{ilm-init}} policy.

{{ilm-init}} may thereby surface:

* Direct errors which would emit when the corresponding {{es}} API was manually executed. The most common examples are [{{ilm-init}} Rollover](elasticsearch://reference/elasticsearch/index-lifecycle-actions/ilm-rollover.md) errors below.
* Indirect issues which would surface after the corresponding {{es}} API was successfully executed but full intention didn't take effect. The most common examples are [{{ilm-init}} Migrate](/reference/elasticsearch/index-lifecycle-actions/ilm-migrate.md) messages below.

In the below guide, we will outline the most common {{ilm-init}} setup misunderstandings and issues.

## Check for {{ilm-init}} issues [check-ilm-for-issues]

{{ilm-init}} runs against each index. It purposely holds an index on a couple of steps for its condition-based and time-based logic. These are mainly `phase/action/step`:

* `hot/rollover/check-rollover-ready` until [{{ilm-init}} Rollover requirements](elasticsearch://reference/elasticsearch/index-lifecycle-actions/ilm-rollover.md#ilm-rollover-options) are met.
* `*/complete/complete` until the index `age` qualifies for the next phase's `min_age`. Refer to [how `min_age` is calculated](#min-age-calculation) below for more information.
* `delete/wait_for_snapshot/wait-for-snapshot` until the [{{ilm-init}} Delete](elasticsearch://reference/elasticsearch/index-lifecycle-actions/ilm-delete.md)'s [SLM policy](/deploy-manage/tools/snapshot-and-restore/create-snapshots.md) is successfully completed for this index.

We will colloquially refer to steps other than these as _transient_ steps where {{ilm-init}} should be applying some operation against the index. Any step may error, but {{es}}'s [Cluster health API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-health-report) will also report `stagnating_indices` for indices which have been in transient steps longer than expected.

When errors occur, {{ilm-init}} updates the [ILM explain API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-ilm-explain-lifecycle) to report:
* the index's active `step` under field `failed_step` and marks `step` as `ERROR`
* flags `is_auto_retryable_error` as `true`
* increments `failed_step_retry_count`

Upon the next {{ilm-init}} index polling

When {{ilm-init}} executes a lifecycle policy, it’s possible for errors to occur while performing the necessary index operations for a step. When this happens, {{ilm-init}} moves the index to an `ERROR` step. If {{ilm-init}} cannot resolve the error automatically, execution is halted until you resolve the underlying issues with the policy, index, or cluster.

See [this video](https://www.youtube.com/watch?v=VCIqkji3IwY) for a walkthrough of troubleshooting current {{ilm-init}} health issues, and [this video](https://www.youtube.com/watch?v=onrnnwjYWSQ) for a walkthrough of troubleshooting historical {{ilm-init}} issues.

For example, you might have a `shrink-index` policy that shrinks an index to four shards once it is at least five days old:

```console
PUT _ilm/policy/shrink-index
{
  "policy": {
    "phases": {
      "warm": {
        "min_age": "5d",
        "actions": {
          "shrink": {
            "number_of_shards": 4
          }
        }
      }
    }
  }
}
```

There is nothing that prevents you from applying the `shrink-index` policy to a new index that has only two shards:

```console
PUT /my-index-000001
{
  "settings": {
    "index.number_of_shards": 2,
    "index.lifecycle.name": "shrink-index"
  }
}
```

After five days, {{ilm-init}} attempts to shrink `my-index-000001` from two shards to four shards. Because the shrink action cannot *increase* the number of shards, this operation fails and {{ilm-init}} moves `my-index-000001` to the `ERROR` step.

You can use the [{{ilm-init}} Explain API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-ilm-explain-lifecycle) to get information about what went wrong:

```console
GET /my-index-000001/_ilm/explain
```

Which returns the following information:

```console-result
{
  "indices" : {
    "my-index-000001" : {
      "index" : "my-index-000001",
      "managed" : true,
      "index_creation_date_millis" : 1541717265865,
      "time_since_index_creation": "5.1d",
      "policy" : "shrink-index",                <1>
      "lifecycle_date_millis" : 1541717265865,
      "age": "5.1d",                            <2>
      "phase" : "warm",                         <3>
      "phase_time_millis" : 1541717272601,
      "action" : "shrink",                      <4>
      "action_time_millis" : 1541717272601,
      "step" : "ERROR",                         <5>
      "step_time_millis" : 1541717272688,
      "failed_step" : "shrink",                 <6>
      "step_info" : {
        "type" : "illegal_argument_exception",  <7>
        "reason" : "the number of target shards [4] must be less that the number of source shards [2]"
      },
      "phase_execution" : {
        "policy" : "shrink-index",
        "phase_definition" : {                  <8>
          "min_age" : "5d",
          "actions" : {
            "shrink" : {
              "number_of_shards" : 4
            }
          }
        },
        "version" : 1,
        "modified_date_in_millis" : 1541717264230
      }
    }
  }
}
```

1. The policy being used to manage the index: `shrink-index`
2. The index age: 5.1 days
3. The phase the index is currently in: `warm`
4. The current action: `shrink`
5. The step the index is currently in: `ERROR`
6. The step that failed to execute: `shrink`
7. The type of error and a description of that error.
8. The definition of the current phase from the `shrink-index` policy


To resolve this, you could update the policy to shrink the index to a single shard after 5 days:

```console
PUT _ilm/policy/shrink-index
{
  "policy": {
    "phases": {
      "warm": {
        "min_age": "5d",
        "actions": {
          "shrink": {
            "number_of_shards": 1
          }
        }
      }
    }
  }
}
```


## Retrying failed lifecycle policy steps [_retrying_failed_lifecycle_policy_steps]

Once you fix the problem that put an index in the `ERROR` step, you might need to explicitly tell {{ilm-init}} to retry the step:

```console
POST /my-index-000001/_ilm/retry
```

{{ilm-init}} subsequently attempts to re-run the step that failed. You can use the [{{ilm-init}} Explain API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-ilm-explain-lifecycle) to monitor the progress.


## Common {{ilm-init}} setting issues [_common_ilm_init_setting_issues]


### How `min_age` is calculated [min-age-calculation]

When setting up an [{{ilm-init}} policy](../../manage-data/lifecycle/index-lifecycle-management/configure-lifecycle-policy.md) or [automating rollover with {{ilm-init}}](../../manage-data/lifecycle/index-lifecycle-management/rollover.md), be aware that `min_age` can be relative to either the rollover time or the index creation time.

If you use [{{ilm-init}} rollover](elasticsearch://reference/elasticsearch/index-lifecycle-actions/ilm-rollover.md), `min_age` is calculated relative to the time the index was rolled over. This is because the [rollover API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-indices-rollover) generates a new index and updates the `age` of the previous index to reflect the rollover time. If the index hasn’t been rolled over, then the `age` is the same as the `creation_date` for the index.

You can override how `min_age` is calculated using the `index.lifecycle.origination_date` and `index.lifecycle.parse_origination_date` [{{ilm-init}} settings](elasticsearch://reference/elasticsearch/configuration-reference/index-lifecycle-management-settings.md).


## Common {{ilm-init}} errors [_common_ilm_init_errors]

Here’s how to resolve the most common errors reported in the `ERROR` step.

### Rollover errors [_common_ilm_init_errors_rollover]

[{{ilm-init}} Rollover](elasticsearch://reference/elasticsearch/index-lifecycle-actions/ilm-rollover.md)

::::{tip}
Problems with rollover aliases are a common cause of errors. Consider using [data streams](/manage-data/data-store/data-streams.md) instead of managing rollover with aliases.
::::


$$$_rollover_alias_x_can_point_to_multiple_indices_found_duplicated_alias_x_in_index_template_z$$$
:::{dropdown} `Rollover alias [x] can point to multiple indices, found duplicated alias [x] in index template [z]` 

The target rollover alias is specified in an index template’s `index.lifecycle.rollover_alias` setting. You need to explicitly configure this alias *one time* when you [bootstrap the initial index](/manage-data/lifecycle/index-lifecycle-management/tutorial-time-series-without-data-streams.md#ilm-gs-alias-bootstrap). The rollover action then manages setting and updating the alias to [roll over](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-indices-rollover#rollover-index-api-desc) to each subsequent index.

Do not explicitly configure this same alias in the aliases section of an index template.

See this [resolving `duplicate alias` video](https://www.youtube.com/watch?v=Ww5POq4zZtY) for an example troubleshooting walkthrough.
:::

$$$_index_lifecycle_rollover_alias_x_does_not_point_to_index_y$$$
:::{dropdown} `index.lifecycle.rollover_alias [x] does not point to index [y]`

Either the index is using the wrong alias or the alias does not exist.

Check the `index.lifecycle.rollover_alias` [index setting](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-indices-get-settings). To see what aliases are configured, use [_cat/aliases](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cat-aliases).

See this [resolving `not point to index` video](https://www.youtube.com/watch?v=NKSe67x7aw8) for an example troubleshooting walkthrough.
:::

$$$_setting_index_lifecycle_rollover_alias_for_index_y_is_empty_or_not_defined$$$
:::{dropdown} `setting [index.lifecycle.rollover_alias] for index [y] is empty or not defined`
The `index.lifecycle.rollover_alias` setting must be configured for the rollover action to work.

Update the index settings to set `index.lifecycle.rollover_alias`.

See this [resolving `empty or not defined` video](https://www.youtube.com/watch?v=LRpMC2GS_FQ) for an example troubleshooting walkthrough.
:::

$$$_alias_x_has_more_than_one_write_index_yz$$$
:::{dropdown} `alias [x] has more than one write index [y,z]`

Only one index can be designated as the write index for a particular alias.

Use the [aliases](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-indices-update-aliases) API to set `is_write_index:false` for all but one index.

See this [resolving `more than one write index` video](https://www.youtube.com/watch?v=jCUvZCT5Hm4) for an example troubleshooting walkthrough.
:::

$$$_index_name_x_does_not_match_pattern_d$$$
:::{dropdown} `index name [x] does not match pattern ^.*-\d+` 

The index name must match the regex pattern `^.*-\d+` for the rollover action to work. The most common problem is that the index name does not contain trailing digits. For example, `my-index` does not match the pattern requirement.

Append a numeric value to the index name, for example `my-index-000001`.

See this [resolving `does not match pattern` video](https://www.youtube.com/watch?v=9sp1zF6iL00) for an example troubleshooting walkthrough.
:::


### {{ilm-init}} Migrate errors [_common_ilm_init_errors_migrate]

[{{ilm-init}} Migrate](/reference/elasticsearch/index-lifecycle-actions/ilm-migrate.md)


$$$ _index_has_a_preference_for_tiers$$$
:::{dropdown} `index has a preference for tiers [xxx] and node does not meet the required [xxx] tier`

If the [allocation explain API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cluster-allocation-explain) returns this error, it indicates that shards cannot be assigned according to the current attribute-based or data tier allocation rules. For detailed guidance on resolving this issue, refer to [Unable to assign shards based on the allocation rule](https://www.elastic.co/docs/troubleshoot/monitoring/unavailable-shards#ec-cannot-assign-shards-on-allocation-rule).  
:::


### General {{ilm-init}} errors [_common_ilm_init_errors_general]

$$$_circuitbreakingexception_x_data_too_large_data_for_y$$$
:::{dropdown} `CircuitBreakingException: [x] data too large, data for [y]`

This indicates that the cluster is hitting resource limits.

Before continuing to set up {{ilm-init}}, you’ll need to take steps to alleviate the resource issues. For more information, see [Circuit breaker errors](circuit-breaker-errors.md).
:::

$$$_high_disk_watermark_x_exceeded_on_y$$$
:::{dropdown} `high disk watermark [x] exceeded on [y]`

This indicates that the cluster is running out of disk space. This can happen when you don’t have {{ilm}} set up to roll over from hot to warm nodes. For more information, see [Fix watermark errors](fix-watermark-errors.md).
:::

$$$_security_exception_action_action_name_is_unauthorized_for_user_user_name_with_roles_role_name_this_action_is_granted_by_the_index_privileges_manage_follow_indexmanageall$$$
:::{dropdown} `security_exception: action [<action-name>] is unauthorized for user [<user-name>] with roles [<role-name>], this action is granted by the index privileges [manage_follow_index,manage,all]`

This indicates the ILM action cannot be executed because the user that ILM uses to perform the action doesn’t have the correct privileges. ILM actions are run as though they are performed by the last user who modified the policy with the privileges that user had at that time. The account used to create or modify the policy must have permissions to perform all operations that are part of that policy. If this error surfaces on system indices, see permissions described in [File-based access recovery](https://www.elastic.co/docs/troubleshoot/elasticsearch/file-based-recovery) to recover.
:::

$$$_policy_policy_name_does_not_exist$$$
:::{dropdown} `policy [<policy-name>] does not exist`

The error occurs because the index is assigned to an ILM policy that does not exist in the cluster. To fix this, you can either [create the missing policy](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-ilm-put-lifecycle) with the required settings or [link the index to an existing ILM policy](https://www.elastic.co/docs/reference/elasticsearch/configuration-reference/index-lifecycle-management-settings#index-lifecycle-name).
:::

