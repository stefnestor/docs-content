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

[{{ilm}}](../../manage-data/lifecycle/index-lifecycle-management.md) executes actions asynchronously against your cluster's indices from logic-based and time-based conditions. These [phase, action, and step operations](/manage-data/lifecycle/index-lifecycle-management/index-lifecycle.md) run sequentially against an index on behalf of the user and their permissions of the time they last edited the [{{ilm-init}} policy](/manage-data/lifecycle/index-lifecycle-management/configure-lifecycle-policy.md).

{{ilm-init}} can thereby surface:

* Direct errors which would also emit if the corresponding {{es}} API was manually executed by that user.
* Indirect configuration issues which would surface after the corresponding {{es}} API successfully executed but the user's intention didn't fully take effect.

This guide covers how to check {{ilm-init}} overall health, investigate individual indices' status, and lists the most common setup questions and issues.

## Check for {{ilm-init}} issues [ilm-check-issues]

We will first outline the symptoms of stuck tasks, then erring tasks, before finally showing the common investigative API commands.

### {{ilm-init}} transient steps [ilm-steps-transient]

{{ilm-init}} purposely holds an index on a couple of steps for its logic-based and time-based conditions. The following [{{ilm-init}} explain API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-ilm-explain-lifecycle) `phase/action/step` combinations wait:

* `hot/rollover/check-rollover-ready` until [{{ilm-init}} Rollover requirements](elasticsearch://reference/elasticsearch/index-lifecycle-actions/ilm-rollover.md#ilm-rollover-options) are met.
* `*/complete/complete` until the index's `age` qualifies for the next phase's `min_age`. Refer to [how `min_age` is calculated](#min-age-calculation) for more information.

We will colloquially refer to steps other than these as _transient_ steps where {{ilm-init}} should be asynchronously applying some operation against the index instead of waiting for a logic-based or time-based condition requirement to be met.

{{ilm-init}} [polls for work on an interval basis](elasticsearch://reference/elasticsearch/configuration-reference/index-lifecycle-management-settings.md) (default `10m`). For more information, refer to [{{ilm-init}} phase transitions](/manage-data/lifecycle/index-lifecycle-management/index-lifecycle.md#ilm-phase-execution). 

Therefore, an index proceeds through its {{ilm-init}} steps as fast as the direct operation's duration in combination with {{ilm-init}}'s poll interval. For example, transient steps whose duration is proportional to their underlying asynchronous operation's duration and can thereby be affected by their [task backlog](/troubleshoot/elasticsearch/task-queue-backlog.md) commonly include:

* `*/migrate/check-migration` monitors the index's [shards' allocation and recoveries](/deploy-manage/distributed-architecture/shard-allocation-relocation-recovery.md).
* `*/*/forcemerge` waits for the index's [force merge](elasticsearch://reference/elasticsearch/index-lifecycle-actions/ilm-forcemerge.md), noting the guide's performance considerations.
* `delete/wait_for_snapshot/wait-for-snapshot` delays until the [{{ilm-init}} Delete](elasticsearch://reference/elasticsearch/index-lifecycle-actions/ilm-delete.md)'s [SLM policy](/deploy-manage/tools/snapshot-and-restore/create-snapshots.md) is successfully completed for the index.

It is not concerning for these transient steps to appear in [{{ilm-init}} explain API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-ilm-explain-lifecycle)'s output. However, investigate investigating if an index does not progress from a particular step for an extended duration. This still might indicate expected behavior based off the nuances of your particular setup and use case, rather than a cluster issue.

### {{ilm-init}} erring steps [ilm-steps-errors]

When errors occur, {{ilm-init}} updates the [{{ilm-init}} explain API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-ilm-explain-lifecycle) to report:

* The index's active `step` under temporary field `failed_step` and marks `step` as `ERROR`.
* Flags the boolean `is_auto_retryable_error`.
* Increments the integer `failed_step_retry_count`.

All erring indices automatically run the [Retry policy API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-ilm-retry) on each {{ilm-init}} polling interval. During automatic or manual retry:

* The `step` will reset to the active step description and `failed_step` will remove.
* The `is_auto_retryable_error` will persist.
* The `failed_step_retry_count` will persist and increment again if another error is encountered.

Non-erring indices will not report the fields `failed_step`, `is_auto_retryable_error`, nor `failed_step_retry_count`. Indices who have recovered from previous errors will also remove these temporary fields. This is why the [{{ilm-init}} explain API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-ilm-explain-lifecycle) allows for flagging `only_errors` to report both indices currently reporting or retrying erring steps:

```console
GET _all/_ilm/explain?human=true&expand_wildcards=all&only_errors=true
```

For troubleshooting, [{{ilm-init}} explain API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-ilm-explain-lifecycle) emits `step_info`. This field will only report when it has further context, such as `message` for information and `reason` for errors.

If {{ilm-init}} cannot automatically resolve the error for this index, execution is halted until the underlying issue with the policy, index, or cluster is resolved. For example, shard migrations might block until [Elastic Cloud Autoscaling](/deploy-manage/autoscaling.md) scales or adds necessary [data tiers](/manage-data/lifecycle/data-tiers.md).

### {{ilm-init}} health [ilm-health-overall]

Now that we know how to identify both transient and erring types of {{ilm-init}} stagnated indices' steps, we will now outline how to poll an overview of {{ilm-init}} health to review them.

{{es}}'s [Cluster health API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-health-report) will report `stagnating_indices` for indices which have been attempting a step longer than expected:

```console
GET _health_report/ilm
```

This report's thresholds are controlled by [Read cluster settings API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cluster-get-settings):

* `health.ilm.max_retries_per_step` (default `100`)
* `health.ilm.max_time_on_action` (default `1d`)
* `health.ilm.max_time_on_step` (default `1d`)

This report consolidates actionable interventions you should consider for your {{ilm-init}} and cluster health.

It is sometimes beneficial to see a high-level summary of all indices' statuses agnostic to if they are stagnated requiring intervention. For example, you can use third-party tool [JQ](https://jqlang.github.io/jq/) against the output of [{{ilm-init}} Explain API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-ilm-explain-lifecycle) saved as `ilm_explain.json` to see an overview of all indices `phase/action/step` statuses:

```bash
$ cat ilm_explain.json | jq -c '.indices[]|select(.managed==true)|.phase+"/"+.action+"/"+.step' | sort | uniq -c | sort -r
```

:::{tip}
For example {{ilm-init}} troubleshooting walkthroughs, refer to

* [Monitoring {{ilm-init}} {{es}} Health](https://www.youtube.com/watch?v=VCIqkji3IwY) for resolving erring steps.
* [{{ilm-init}} History Index](https://www.youtube.com/watch?v=onrnnwjYWSQ) for an explanation of step sequences and how to review historical index statuses.
:::

### Troubleshooting {{ilm-init}} for an index [ilm-health-index]

Let's demonstrate troubleshooting {{ilm-init}} for a newly created index. For example, we might have a `shrink-index` policy that [Shrinks](elasticsearch://reference/elasticsearch/index-lifecycle-actions/ilm-shrink.md) an index to four shards once it is at least five days old:

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

Now when we [Create an index](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-indices-create) `my-index-000001` that has only two primary shards, we are able to [apply the {{ilm-init}} policy](/manage-data/lifecycle/index-lifecycle-management/policy-apply.md) `shrink-index`:

```console
PUT /my-index-000001
{
  "settings": {
    "index.number_of_shards": 2,
    "index.lifecycle.name": "shrink-index"
  }
}
```

After five days, {{ilm-init}} attempts to run the [Shrink index API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-indices-shrink) against index `my-index-000001` from two shards to four shards. Because the shrink action cannot *increase* the number of shards, this operation fails and {{ilm-init}} moves `my-index-000001` to the `step` of `ERROR`.

We can use the [{{ilm-init}} Explain API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-ilm-explain-lifecycle) to get information about what went wrong:

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


To resolve this, we could update the policy to shrink the index to a single shard after 5 days:

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

Once we fix the problem, we can wait for {{ilm-init}}'s poll interval to automatically retry the index's `ERROR` step or can apply the [Retry policy API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-ilm-retry) to run it on demand:

```console
POST /my-index-000001/_ilm/retry
```

{{ilm-init}} subsequently attempts to re-run the step that failed. You can use the [{{ilm-init}} Explain API]({{es-apis}}operation/operation-ilm-explain-lifecycle) to monitor the progress.

## Frequently Asked Questions [ilm-faq]

We have collected the most frequently asked questions here. If your question isn’t answered here or in [our {{ilm-init}} guide](/manage-data/lifecycle/index-lifecycle-management.md), then [contact us](/troubleshoot/index.md#contact-us).

### How `min_age` is calculated [min-age-calculation]

When setting up an [{{ilm-init}} policy](../../manage-data/lifecycle/index-lifecycle-management/configure-lifecycle-policy.md) or [automating rollover with {{ilm-init}}](../../manage-data/lifecycle/index-lifecycle-management/rollover.md), be aware that `min_age` can be relative to either the rollover time or the index creation time.

If you use [{{ilm-init}} rollover](elasticsearch://reference/elasticsearch/index-lifecycle-actions/ilm-rollover.md), `min_age` is calculated relative to the time the index was rolled over. This is because the [rollover API]({{es-apis}}operation/operation-indices-rollover) generates a new index and updates the `age` of the previous index to reflect the rollover time. If the index hasn’t been rolled over, then the `age` is the same as the `creation_date` for the index.

You can override how `min_age` is calculated using the `index.lifecycle.origination_date` and `index.lifecycle.parse_origination_date` [{{ilm-init}} settings](elasticsearch://reference/elasticsearch/configuration-reference/index-lifecycle-management-settings.md).

### Steps proceed sequentially [ilm-sequential-steps]

{{ilm-init}} does not skip steps due to logic-based nor time-based conditions. It will proceed through all steps [in the enabled action's order](/manage-data/lifecycle/index-lifecycle-management/index-lifecycle.md#phases-availability). For example, this means it's possible for an index stagnated at `phase/action/step` of `warm/migrate/check-migration` to surpass its expected deletion time. This is an example of why it's important to review and resolve {{ilm-init}} errors to maintain a healthy cluster. For more information, refer to [{{ilm-init}} phase transitions](/manage-data/lifecycle/index-lifecycle-management/index-lifecycle.md#ilm-phase-transitions).

### Policy changes apply forward [ilm-policy-changes]

Upon entering a phase, an index caches the {{ilm-init}} policy's current definition of it. For more information, refer to [phase execution](/manage-data/lifecycle/index-lifecycle-management/index-lifecycle.md#ilm-phase-execution). This enables {{ilm-init}} to protect the index from policy changes which might cause data corruption. 

As outlined in [how changes are applied](/manage-data/lifecycle/index-lifecycle-management/policy-updates.md#ilm-apply-changes), {{ilm-init}} will update an index's `phase_execution` with safe policy updates, but updates deemed unsafe to apply backwards will only apply forwards to indices as they enter the phase. 

You might encounter a situation where policy changes need to be backwards applied to stagnant indices anyway. There is no mechanism to one-off run an {{ilm-init}} step as this might expose indices to corruption. Instead, you should manually apply the delta of changes to indices as needed.

In rare circumstances, policy changes might induce stagnant indices requiring the [Move to an {{ilm-init}} step API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-ilm-move-to-step) to resolve. This is considered an advanced API and we recommend [contacting us](/troubleshoot/index.md#contact-us) with an [{{es}} diagnostic](/troubleshoot/elasticsearch/diagnostic.md) before intervening.

## Common {{ilm-init}} errors [ilm-errors]

Here’s how to resolve the most common errors reported in the `ERROR` step.

### Rollover errors [ilm-errors-rollover]

::::{tip}
Problems with rollover aliases are a common cause of errors. You should consider using [data streams](/manage-data/data-store/data-streams.md) instead of managing rollover with aliases.
::::

Here's how to resolve the most common [{{ilm-init}} Rollover](elasticsearch://reference/elasticsearch/index-lifecycle-actions/ilm-rollover.md) errors:

:::{dropdown} `Rollover alias [x] can point to multiple indices, found duplicated alias [x] in index template [z]`
:name: _rollover_alias_x_can_point_to_multiple_indices_found_duplicated_alias_x_in_index_template_z

The target rollover alias is specified in an index template’s `index.lifecycle.rollover_alias` setting. You need to explicitly configure this alias *one time* when you [bootstrap the initial index](/manage-data/lifecycle/index-lifecycle-management/tutorial-time-series-without-data-streams.md#ilm-gs-alias-bootstrap). The rollover action then manages setting and updating the alias to [roll over]({{es-apis}}operation/operation-indices-rollover#rollover-index-api-desc) to each subsequent index.

Do not explicitly configure this same alias in the aliases section of an index template.

Refer to [resolving `duplicate alias` video](https://www.youtube.com/watch?v=Ww5POq4zZtY) for an example troubleshooting walkthrough.
:::

:::{dropdown} `index.lifecycle.rollover_alias [x] does not point to index [y]`
:name: _index_lifecycle_rollover_alias_x_does_not_point_to_index_y

Either the index is using the wrong alias or the alias does not exist.

Check the `index.lifecycle.rollover_alias` [index setting]({{es-apis}}operation/operation-indices-get-settings). To see what aliases are configured, use [_cat/aliases]({{es-apis}}operation/operation-cat-aliases).

Refer to [resolving `not point to index` video](https://www.youtube.com/watch?v=NKSe67x7aw8) for an example troubleshooting walkthrough.
:::

:::{dropdown} `setting [index.lifecycle.rollover_alias] for index [y] is empty or not defined`
:name: _setting_index_lifecycle_rollover_alias_for_index_y_is_empty_or_not_defined

The `index.lifecycle.rollover_alias` setting must be configured for the rollover action to work.

Update the index settings to set `index.lifecycle.rollover_alias`.

Refer to [resolving `empty or not defined` video](https://www.youtube.com/watch?v=LRpMC2GS_FQ) for an example troubleshooting walkthrough.
:::

:::{dropdown} `alias [x] has more than one write index [y,z]`
:name: _alias_x_has_more_than_one_write_index_yz

Only one index can be designated as the write index for a particular alias.

Use the [aliases]({{es-apis}}operation/operation-indices-update-aliases) API to set `is_write_index:false` for all but one index.

Refer to [resolving `more than one write index` video](https://www.youtube.com/watch?v=jCUvZCT5Hm4) for an example troubleshooting walkthrough.
:::

:::{dropdown} `index name [x] does not match pattern ^.*-\d+` 
:name: _index_name_x_does_not_match_pattern_d

The index name must match the regex pattern `^.*-\d+` for the rollover action to work. The most common problem is that the index name does not contain trailing digits. For example, `my-index` does not match the pattern requirement.

Append a numeric value to the index name, for example `my-index-000001`.

Refer to [resolving `does not match pattern` video](https://www.youtube.com/watch?v=9sp1zF6iL00) for an example troubleshooting walkthrough.
:::


### {{ilm-init}} Migrate errors [ilm-errors-migrate]

The following errors usually surface during shard recovery. This primarily occurs during [{{ilm-init}} Migrate](elasticsearch://reference/elasticsearch/index-lifecycle-actions/ilm-migrate.md), but can also surface for example during [{{ilm-init}} Searchable Snapshots](elasticsearch://reference/elasticsearch/index-lifecycle-actions/ilm-searchable-snapshot.md). As an asynchronous operation, the {{ilm-init}}-level message will frequently only report a symptomatic error. Refer to [Troubleshooting Allocation](/troubleshoot/elasticsearch/cluster-allocation-api-examples.md) for Allocation-level causal error examples.


:::{dropdown} `index has a preference for tiers [xxx] and node does not meet the required [xxx] tier`
:name: _index_has_a_preference_for_tiers

If the [allocation explain API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cluster-allocation-explain) returns this error, it indicates that shards cannot be assigned according to the current attribute-based or data tier allocation rules. For detailed guidance on resolving this issue, refer to [Unable to assign shards based on the allocation rule](https://www.elastic.co/docs/troubleshoot/monitoring/unavailable-shards#ec-cannot-assign-shards-on-allocation-rule).  
:::

### General {{ilm-init}} errors [ilm-errors-general]

The following errors can surface on any {{ilm-init}} step.

:::{dropdown} `CircuitBreakingException: [x] data too large, data for [y]`
:name: _circuitbreakingexception_x_data_too_large_data_for_y

This indicates that the cluster is hitting resource limits.

Before continuing to set up {{ilm-init}}, you’ll need to take steps to alleviate the resource issues. For more information, see [Circuit breaker errors](circuit-breaker-errors.md).
:::

:::{dropdown} `high disk watermark [x] exceeded on [y]`
:name: _high_disk_watermark_x_exceeded_on_y

This indicates that the cluster is running out of disk space. This can happen when you don’t have {{ilm}} set up to roll over from hot to warm nodes. For more information, see [Fix watermark errors](fix-watermark-errors.md).

:::{dropdown} `security_exception: action [<action-name>] is unauthorized for user [<user-name>] with roles [<role-name>], this action is granted by the index privileges [manage_follow_index,manage,all]`
:name: _security_exception

This indicates the {{ilm-init}} action cannot be executed because the user that {{ilm-init}} uses to perform the action doesn’t have the correct privileges. {{ilm-init}} actions are run as though they are performed by the last user who modified the policy with the privileges that user had at that time. The account used to create or modify the policy must have permissions to perform all operations that are part of that policy. If this error surfaces on system indices, see permissions described in [File-based access recovery](https://www.elastic.co/docs/troubleshoot/elasticsearch/file-based-recovery) to recover.
:::

:::{dropdown} `policy [<policy-name>] does not exist`
:name: _policy_policy_name_does_not_exist

The error occurs because the index is assigned to an {{ilm-init}} policy that does not exist in the cluster. To fix this, you can either [create the missing policy](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-ilm-put-lifecycle) with the required settings or [link the index to an existing {{ilm-init}} policy](https://www.elastic.co/docs/reference/elasticsearch/configuration-reference/index-lifecycle-management-settings#index-lifecycle-name).
:::
