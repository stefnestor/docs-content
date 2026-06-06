---
navigation_title: Fix date timestamps
description: ""
type: troubleshooting
applies_to:
  stack: ga
products:
  - id: elasticsearch
---

# Fix date timestamps [fix-date-timestamps]

{{es}} will accept any valid past, present, or future date for its [date fields](elasticsearch://reference/elasticsearch/mapping-reference/date.md) as long as the value satisfies that field's [date `format` setting](elasticsearch://reference/elasticsearch/mapping-reference/mapping-date-format.md).

Users should ensure stored date fields are valid. {{es}} accepting a value does not reflect its accuracy. The following are not guaranteed to be problematic for your setup but are common client-side data quality issues that users need to consider to ensure their timestamp's accuracy:

* Operating system timezones
* Variances in date formatting
* Unexpected values such as:
    * future dates
    * default dates, such as the `1970` [unix epoch](https://en.wikipedia.org/wiki/Unix_time)
    * negative dates
    * time-bucketed dates
    * truncated strings

We recommend resolving timestamp data quality issues as close to problem source as possible. The following guide outlines symptoms which can occur due to and how to account for timestamp data quality issues. Since timestamp data quality issues can commonly affect the performance of scheduled tasks which search `now-X`, our outline will focus examples on checking for unexpected future dates within the `@timestamp` date field. 

## Symptoms [fix-date-timestamps-symptoms]

Timestamp data quality issues can strain resources and cause unexpected search results.

The following are common locations where users first notice issues:

* [Discover](/explore-analyze/discover.md)
* [Dashboards](/explore-analyze/dashboards.md)
* Security's [Detections and alerts](/solutions/security/detect-and-alert.md)
* Observability's [Incident management alerts](/solutions/observability/incident-management/alerting.md)
* [{{kib}} alerting](/explore-analyze/alerting/alerts.md)
* [Watchers](/explore-analyze/alerting/watcher.md)

The following are symptoms that users commonly first notice:

* There is a full [search queue backlog](/troubleshoot/elasticsearch/task-queue-backlog.md#diagnose-task-queue-thread-pool) on later [data tiers](/manage-data/lifecycle/data-tiers.md) but not blocking on `data_hot`.
* The `data_frozen` data tier experiences ongoing [high CPU usage](/troubleshoot/elasticsearch/high-cpu-usage.md).
* [Slow logs](elasticsearch://reference/elasticsearch/index-settings/slow-log.md) report events for `data_cold` or `data_frozen` on short, periodic intervals.
* [{{kib}}'s Inspect](https://www.elastic.co/blog/troubleshooting-guide-common-issues-kibana-discover-load#3.-load-search) reports search results from unexpected indices.
* Indices created based on [{{ls}}'s date math syntax](https://www.elastic.co/docs/reference/logstash/plugins/plugins-outputs-elasticsearch#plugins-outputs-elasticsearch-index) or {{es}}'s [Date index name processor](elasticsearch://reference/enrich-processor/date-index-name-processor.md) reference dates far into the past or future.

Performance incidents can frequently be avoided by following [best practices](#fix-date-timestamps-recommendations) even if underlying timestamp data quality issues remain.

### Example [fix-date-timestamps-example]

Here's an example to demonstrate how timestamp data quality issues can affect the search performance of your cluster.

As setup, let's say a single on-prem host has a bad time configuration logging a future date that is one year into the future. This host's `@timestamp` date field will report data a year ahead of other hosts logging similar data.

For our example, all these hosts' data ingests into a 5 [primaries](elasticsearch://reference/elasticsearch/index-settings/index-modules.md#index-number-of-shards) {{es}} [data stream](/manage-data/data-store/data-streams.md). This data stream has an [Index lifecycle management policy](/manage-data/lifecycle/index-lifecycle-management.md) that guarantees it [rolls over indices](elasticsearch://reference/elasticsearch/index-lifecycle-actions/ilm-rollover.md) at least once a day and that [migrates data to frozen tier](/manage-data/lifecycle/index-lifecycle-management/index-lifecycle.md#ilm-phase-actions) after 15 days. After 30 days, there are 150 shards with half hosted in frozen tier.

This data stream is powering a [dashboard](/explore-analyze/dashboards.md). The dashboard uses a [data view](/explore-analyze/find-and-organize/data-views.md) with the `@timestamp` field as its designated date field.

If this misconfigured host was not ingesting data those 30 days, when a user selects a `now-15m` [time window](/explore-analyze/dashboards/using.md#_set_a_time_range), you could normally expect:

1. The [search pre-filter]({{es-apis}}operation/operation-msearch) could narrow into only the latest backing index's 5 shards.
1. The data [would be read](/deploy-manage/distributed-architecture/reading-and-writing-documents.md#_basic_read_model) off of only these shards and remaining search [queries](/explore-analyze/query-filter.md) and [aggregations](/explore-analyze/query-filter/aggregations.md) would run off that subset of data.
1. The Dashboard's [Inspect](/troubleshoot/observability/inspect.md) would report 5 shards searched and 145 shards `skipped`.

However, if this misconfigured host was ingesting data those 30 days, then you could expect:

1. The pre-filter cannot filter-out any backing indices, so will allow search against all 150 shards backing this data stream.
1. Half of the shards are in the `data_frozen` data tier, which is intended for [rarely queried data](/manage-data/lifecycle/data-tiers.md). This is because:
    * Frozen tier hardware profile is usually provisioned to be proportioned for low CPU to high data, so searches run slower. 
    * These indices are [partially-mounted searchable snapshots](/deploy-manage/tools/snapshot-and-restore/searchable-snapshots.md) so searches can run slower as data needs fetched out of snapshot repository.
1. The cluster will search all 150 shards for timestamps within the desired window. This resource usage can happen even if no documents end up qualifying for the requested time range.
1. The Dashboard's [Inspect](/troubleshoot/observability/inspect.md) would report 150 shards searched and 0 `skipped`.

From this you can see that the search is more computationally expensive and can potentially return different results based off the host having misconfigured timestamps. Next, we will discuss how to investigate the scope of the data quality issue.

## Investigate

Timestamp data quality issues can be hard to notice if they're not actively causing performance strain. It can require familiarity with the seasonal patterns of the data. 

A common sign to look for is date values far into the past or future. You can check for future dates with the following options:

* To review [partially-mounted searchable snapshots](/deploy-manage/tools/snapshot-and-restore/searchable-snapshots.md#searchable-snapshot-mount-storage-options) and their `@timestamp` date field only, you can [read the cluster state]({{es-apis}}operation/operation-cluster-state):

    ```console
    GET _cluster/state?filter_path=metadata.indices.*.timestamp_range
    ```

    You can use third-party tools such as [JQ](https://jqlang.github.io/jq/manual/) to filter and format these results. For example to see a list of indices who's maximum timestamp is in the future from the time the command is ran:

    ```sh
    cat cluster_state.json | jq -cMr '.metadata.indices| to_entries| sort_by(.key)| .[]| .value.timestamp_range as $ts| select($ts.min)| {min:($ts.min/1000.0 | todate),max:($ts.max/1000.0 | todate), index:.key}' | jq -r --arg now "$(date -u +"%Y-%m-%dT%H:%M:%SZ")" 'select(.max > $now)'
    ```

* To list the top 200 [aggregated](/explore-analyze/query-filter/aggregations.md) indices by document count of having timestamps in the future from the time the command is ran:

    ```console
    GET */_search
    { "size": 0,
      "aggs": { "0": {
        "terms": {
          "field": "_index",
          "order": { "_count": "desc" },
          "size": 200
        }}
      },
      "query": {
        "bool": {
          "filter": [ { "range": { "@timestamp": { "gte": "now" }}}]
        }
      }
    }
    ``` 

* You can list individual indices' minimum and maximum timestamps. This can be expensive, depending on your search target scope and the hardware profiles of nodes hosting related shards.

    ```console
    GET my_datastream/_search
    { "size": 0,
      "aggs": { "2": {
        "aggs": {
          "min": {"min": {"field": "@timestamp"} },
          "max": {"max": {"field": "@timestamp"} }
        },
        "terms": {
          "field": "_index",
          "order": {"_key": "asc"},
          "size": 200
        }
      }}
    }
    ```

If future dates are found, you will likely want to resolve this data quality concern. If so, it can help to check for potential seasonal patterns in their distributions in order for your team to decide how to proceed:

```console
GET my_datastream/_search?filter_path=aggregations
{ "size": 0,
  "query": {
    "bool": {
      "filter": [ { "range": { "@timestamp": { "gte": "now" }}}]
    }
  },
  "aggs": {
    "time_buckets": {
      "auto_date_histogram": {
        "field": "@timestamp",
        "buckets": 30,
        "format": "yyyy-MM-dd"
      }
    }
  }
}
```

Once you have investigated the scope and source of your timestamp data quality issue, you can determine how your team would like to resolve it. On the rest of the page, we will outline [best practices](#fix-date-timestamps-recommendations) and common [cleanup](#fix-date-timestamps-cleanup) options.

## Best practices [fix-date-timestamps-recommendations]

You can consider the following options to minimize the impact of timestamp data quality issues. You should consider these for all scheduled tasks.

### Default date columns [fix-date-timestamps-defaults]

[Time series data](/manage-data/lifecycle.md) is frequently searched by its date fields. The most common date field used is [`@timestamp`](https://www.elastic.co/docs/reference/ecs/ecs-base). By default, this field's value reflects when the event originated as reported by the source. This is the most common date view users expect to use to search their data, so is the default date field when creating a [data view](/explore-analyze/find-and-organize/data-views.md). Data views are how Discover and Dashboard objects [resolve]({{es-apis}}operation/operation-indices-resolve-index) to and search data.

When users are not present, you should instead consider searching off of the [`event.ingested`](https://www.elastic.co/docs/reference/ecs/ecs-event) date field. By default, this field's value reflects when the event ingested into the cluster. Refer to [this troubleshooting guide](/troubleshoot/elasticsearch/troubleshoot-ingest-pipelines.md) to add it to your data with a custom [ingest pipeline](/manage-data/ingest/transform-enrich/ingest-pipelines.md) if it is not already populating. To avoid complications from ingestion lag, you should first consider `event.ingested` for all scheduled tasks, such as:

* [Transforms](/explore-analyze/transforms.md)
* [Machine Learning](/explore-analyze/machine-learning.md)
* [{{kib}} alerting](/explore-analyze/alerting/alerts.md)
* [Watchers](/explore-analyze/alerting/watcher.md)

{{kib}} [solutions](/solutions/index.md) can encourage this, like in [Security's data sources](/solutions/security/detect-and-alert/set-rule-data-sources.md#best-practices), or automatically choose this on your behalf, like in [Observability's rules](/solutions/observability/incident-management/create-manage-rules.md#observability-create-manage-rules-observability-rules).

For Security's [Detection rules](/solutions/security/detect-and-alert.md), you might also consider enabling the **Do not use @timestamp as a fallback timestamp field** [advanced setting](/solutions/security/detect-and-alert/common-rule-settings.md#rule-ui-advanced-params).

### {{kib}} data source settings [fix-date-timestamps-flags]

You might consider enabling {{kib}} [Advanced Settings](kibana://reference/advanced-settings.md) to avoid `data_cold,data_frozen` [data tiers](/manage-data/lifecycle/data-tiers.md) with the following settings:

* `data_views:fields_excluded_data_tiers` for all [Data views](/explore-analyze/find-and-organize/data-views.md):
* `observability:searchExcludedDataTiers` for the [Observability solution](/solutions/observability.md)
* For the [Security solution](/solutions/security.md):
    * `securitySolution:excludedDataTiersForRuleExecution`
    * `securitySolution:excludeColdAndFrozenTiersInPrevalence`
    * `securitySolution:excludeColdAndFrozenTiersInAnalyzer`

    :::{tip}
    Refer also to [](/solutions/security/detect-and-alert/set-rule-data-sources.md#exclude-cold-frozen-tier) for best practices.
    :::

### Data tier filters [fix-date-timestamps-tier]

The {{es}} search pre-filter can filter in or out of [data tiers](/manage-data/lifecycle/data-tiers.md) with a [boolean query](elasticsearch://reference/query-languages/query-dsl/query-dsl-bool-query.md) DSL. Filtering with [query string query](elasticsearch://reference/query-languages/query-dsl/query-dsl-query-string-query.md) is insufficient. For example, you could filter out `data_cold,data_frozen` like:

```json
{
   "bool":{
      "must_not":{
         "terms":{
            "_tier":[ "data_cold", "data_frozen" ]
         }
      }
   }
}
```

## Cleanup [fix-date-timestamps-cleanup]

You can consider the following cleanup options for your timestamp data quality issues. 

### Delete problematic data [fix-date-timestamps-cleanup-delete]

If your team determines it is best to remove invalid data, then you can delete it either by:

* [Deleting the index]({{es-apis}}operation/operation-indices-delete) or [deleting the data stream]({{es-apis}}operation/operation-indices-delete-data-stream). For [searchable snapshot indices](/deploy-manage/tools/snapshot-and-restore/searchable-snapshots.md), decide if you will [delete their associated snapshots](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-snapshot-delete).

* Running [Delete by query]({{es-apis}}operation/operation-delete-by-query) against the index to remove problematic data. For example, to remove documents with future dates:

    ```console
    POST my_index/_delete_by_query
    {
      "query": {
        "range": {
          "@timestamp": {
            "gt": "now"
          }
        }
      }
    }
    ```

### Modify problematic data [fix-date-timestamps-cleanup-modify]

:::{tip}
To modify documents within a [searchable snapshot index](/deploy-manage/tools/snapshot-and-restore/searchable-snapshots.md), you must first [restore it to a normal index](/manage-data/lifecycle/data-tiers/manage-data-tiers-ech-ece.md#searchable-snapshot-data-tier).
:::

The following is an example flow which users commonly consider to modify invalid data by updating it to current time:

1. Create an [Ingest pipeline](/manage-data/ingest/transform-enrich/ingest-pipelines.md) which modifies the `@timestamp` date field to the current timestamp

    ```console
    PUT _ingest/pipeline/update_date
    {
      "processors": [
        {
          "rename": {
            "description": "(Optional) Cache the previous timestamp in a new field",
            "field": "@timestamp",
            "target_field": "old_timestamp"
          }
          ,
          "set": {
            "description": "Override the @timestamp value to the ingested time",
            "field": "_source.@timestamp",
            "value": "{{_ingest.timestamp}}"
          }
        }
      ]
    }
    ```

1. Run the data over that pipeline to modify the value, either 

    * Across the entire index by [reindexing]({{es-apis}}operation/operation-reindex) into a new index.

        ```console
        POST _reindex
        {
          "source": {
            "index": ["my-index-000001", "my-index-000002"]
          },
          "dest": {
            "index": "my-new-index-000001",
            "pipeline": "update_date"
          }
        }
        ```

    * Targeting specific documents by [updating by query]({{es-apis}}operation/operation-update-by-query) within the existing index.

        ```console
        POST my_index/_update_by_query?pipeline=update_date
        {
          "query": {
            "range": {
              "@timestamp": {
                "gt": "now"
              }
            }
          }
        }
        ```
