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

Users should ensure stored date fields are valid. Just because {{es}} can accept a value, does not reflect its accuracy. The following are not guaranteed to be problematic for your setup but are common client-side data quality issues that users need to review to ensure their timestamp's accuracy:

* Operating system timezones
* Variances in date formatting
* Unexpected values such as:
    * future dates
    * default dates, such as the [unix epoch](https://en.wikipedia.org/wiki/Unix_time) in 1970
    * negative dates
    * bucketed or truncated strings

The following guide outlines symptoms which can surface due to timestamp data quality issues and how to resolve them.

## Symptoms [fix-date-timestamps-symptoms]

Timestamp data quality issues can strain resources and cause unexpected search results. The following are common locations where and symptoms in which users first notice their timestamp data quality issues:

* locations
    * [Discover](/explore-analyze/discover.md)
    * [Dashboards](/explore-analyze/dashboards.md)
    * Security's [Detections and alerts](/solutions/security/detect-and-alert.md)
    * Observability's [Incident management alerts](/solutions/observability/incident-management/alerting.md)
    * [Kibana alerting](/explore-analyze/alerting/alerts.md)
* symptoms
    * There is a [search queue backlog](/troubleshoot/elasticsearch/task-queue-backlog.md#diagnose-task-queue-thread-pool) on later [data tiers](/manage-data/lifecycle/data-tiers.md) but not on `data_hot`.
    * The `data_frozen` data tier experiences [high CPU usage](/troubleshoot/elasticsearch/high-cpu-usage.md).
    * [Slow logs](elasticsearch://reference/elasticsearch/index-settings/slow-log.md) report logs for `data_cold` or `data_frozen` on short, periodic intervals.
    * [Kibana's Inspect](https://www.elastic.co/blog/troubleshooting-guide-common-issues-kibana-discover-load#3.-load-search) reports search results from unexpected indices.
    * Indices created based off [{{ls}}'s date math dynamic syntax](https://www.elastic.co/docs/reference/logstash/plugins/plugins-outputs-elasticsearch#plugins-outputs-elasticsearch-index) or {{es}}'s [Date index name processor](/reference/enrich-processor/date-index-name-processor.md) reference dates far into the past or future.

Performance incidents can frequently be avoided by being careful with your [default selected date column](#[fix-date-timestamps-defaults]) for scheduled tasks.

### Example [fix-date-timestamps-example]

Let's demonstrate how timestamp data quality issues can affect the search performance of your cluster.

As setup, let's say a single on-prem host has a bad time configuration logging a future date that is one year into the future. This host's `@timestamp` date field will report data a year ahead of other hosts logging similar data.

Let's say all these hosts' data ingests into a 5 [primaries](elasticsearch://reference/elasticsearch/index-settings/index-modules.md#index-number-of-shards) {{es}} [data stream](/manage-data/data-store/data-streams.md). This data stream has an [Index lifecycle management policy](/manage-data/lifecycle/index-lifecycle-management.md) that guaranteed [rolls over](elasticsearch://reference/elasticsearch/index-lifecycle-actions/ilm-rollover.md) at least once a day and that [ages this data into the frozen tier](/manage-data/lifecycle/index-lifecycle-management/index-lifecycle.md#ilm-phase-actions) after 15 days. By the end of 30 days there could be 150 shards, with roughly half those allocated onto the frozen tier.

Let's pretend this data stream is powering a [dashboard](/explore-analyze/dashboards.md). The dashboard uses a [Data view](/explore-analyze/find-and-organize/data-views.md) using the `@timestamp` date field. 

Before this misconfigured host ingests data, when a user selects a `now-15m` [time window](/explore-analyze/dashboards/using.md#_set_a_time_range), you could normally expect:

1. The [search pre-filter]({{es-apis}}operation/operation-msearch) could narrow into only the latest backing index's 5 shards.
1. The data [would be read](/deploy-manage/distributed-architecture/reading-and-writing-documents.md#_basic_read_model) off of only these shards. 
1. The remaining search [queries](/explore-analyze/query-filter.md) and [aggregations](/explore-analyze/query-filter/aggregations.md) would run off that subset of data.
1. The Dashboard's [Inspect](/troubleshoot/observability/inspect.md) would report 5 shards searched and 145 `skipped`.

However, if this one misconfigured host has been ingesting data the whole month, then you could expect:

1. The pre-filter cannot narrow into any backing indices, so will match to all 150 data stream backing shards.
1. Half of the shards are in the `data_frozen` data tier, which is intended for [rarely queried data](/manage-data/lifecycle/data-tiers.md). This is because:
    * Frozen tier hardware is usually provisioned to be proportioned for low CPU for high data, so searches normally run slower. 
    * These indices are [partially-mounted searchable snapshots](/deploy-manage/tools/snapshot-and-restore/searchable-snapshots.md) so searches are typically slower when data must be fetch out of snapshot repository.
1. The cluster will search every shard for matching timestamps. This resource usage can happen even if no documents end up qualifying for the requested time range.
1. The resulting matching documents could surface more data or could end up with the same as the data as found when the host was not ingesting bad timestamps.
1. The Dashboard's [Inspect](/troubleshoot/observability/inspect.md) would report 150 shards searched and 0 `skipped`.

From this you can see that the search is more computatively expensive and can potentially return different results based off the host having timestamps misconfigured.

## Best practices 

### Default date columns [fix-date-timestamps-defaults]

[Time series data](/manage-data/lifecycle.md) is frequently searched by its date fields.

The most common date field used is [`@timestamp`](https://www.elastic.co/docs/reference/ecs/ecs-base). By default, this field's value reflects when the event originated as reported by the source. This is the most common date view users expect to search their data, so is the default date field when creating [Data views](/explore-analyze/find-and-organize/data-views.md), which are used in Discover and Dashboards as examples.

When users are not present, you should instead consider searching off of the [`event.ingested`](https://www.elastic.co/docs/reference/ecs/ecs-event) date field. By default, this field's value reflects when the event ingested into the cluster. Refer to [this troubleshooting guide](/troubleshoot/elasticsearch/troubleshoot-ingest-pipelines.md) to add it to your data set via [ingest pipeline](/manage-data/ingest/transform-enrich/ingest-pipelines.md) if it is not already populating. You should consider `event.ingested` first for all scheduled tasks, to avoid complications of ingestion lag, such as:

* [Transforms](/explore-analyze/transforms.md)
* [Machine Learning](/explore-analyze/machine-learning.md)
* [Kibana alerting](/explore-analyze/alerting/alerts.md)

Kibana Solutions may encourage this, like in [Security's data sources](/solutions/security/detect-and-alert/set-rule-data-sources.md#best-practices), or automatically choose this on your behalf, like in [Observability's built-in rule types](/solutions/observability/incident-management/create-manage-rules.md#observability-create-manage-rules-observability-rules).









STEF_TO_HERE_🙋‍♀️













## Problem Box

If you suspect a cluster, data stream or index has unexpected, future timestamps, you can:

A) globally check cluster state
B) globally check for documents with future dates
C) individually check an index's time range

### A) Cluster State

Best-effort, check all indices' `@timestamp` min/max timestamps for versions ≥7.17 by storing [Cluster State](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-state.html) command ...

```console
GET _cluster/state?filter_path=metadata.indices.*.timestamp_range
```

... into a local file called `cluster_state.json`. Then using your favorite tool to parse these epoch dates (example is via [JQ](https://jqlang.github.io/jq/manual/)), you can see a table of indices and min/max timestamps

```sh
cat cluster_state.json | jq -cMr '.metadata.indices| to_entries| sort_by(.key)| .[]| .value.timestamp_range as $ts| select($ts.min)| {min:($ts.min/1000.0 | todate),max:($ts.max/1000.0 | todate), index:.key}' | jq -r --arg now "$(date -u +"%Y-%m-%dT%H:%M:%SZ")" 'select(.max > $now)'
```

For example, reading a successful parsing of the above, we see on 2023 June 16 that the first Frozen index has max data from 2023 February but the second has data up to _2024_ February. 

```sh
{"min":"2023-02-20T11:13:28Z","max":"2023-02-21T01:54:56Z","index":"partial-.ds-logs-zscaler.zscalernss-web-default-2023.02.20-000011"}
{"min":"2023-02-20T18:00:43Z","max":"2024-02-21T13:36:50Z","index":"partial-.ds-logs-zscaler.zscalernss-web-default-2023.02.21-000012"}
```
- Note:  This will convert the json text into tabular format - new code snippet `| jq  -r '"\(.index) \t\(.min) \t\(.max)"'`
```sh
cat cluster_state.json | jq -cMr '.metadata.indices| to_entries| sort_by(.key)| .[]| .value.timestamp_range as $ts| select($ts.min)| {min:($ts.min/1000.0 | todate),max:($ts.max/1000.0 | todate), index:.key}' | jq -r --arg now "$(date -u +"%Y-%m-%dT%H:%M:%SZ")" 'select(.max > $now)' | jq  -r '"\(.index) \t\(.min) \t\(.max)"'
```
- output
```sh
partial-.ds-logs-zscaler.zscalernss-web-default-2023.02.20-000011   2023-02-20T11:13:28Z  2023-02-21T01:54:56Z
partial-.ds-logs-zscaler.zscalernss-web-default-2023.02.21-000012   2023-02-20T18:00:43Z  2024-02-21T13:36:50Z
```

### B) Search for future docs 

Use an [Aggregation Search](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html) to find all indices with future from `now` dates on their `@timestamp` fields ...

(Please notice that this query is only returning future dates greater than today - if overlap is in the past, then it would not return any documents and we should use approach C)

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
      "must": [],
      "filter": [ { "range": { "@timestamp": { "gte": "now" }}}]
    }
  }
}
``` 

### C) Individual Check

Use an [Aggregation Search](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html) to confirm if a particular index's date ranges. Targeting an example [Data Stream](https://www.elastic.co/guide/en/elasticsearch/reference/current/data-streams.html)'s backing index pattern `.ds-comments-*` for date field `@timestamp` ... 

```console
GET .ds-comments-*/_search?filter_path=aggregations
{ "size": 0,
  "aggs": { "2": {
    "aggs": {
      "first_time": {"min": {"field": "@timestamp"} },
      "last_time": {"max": {"field": "@timestamp"} }
    },
    "terms": {
      "field": "_index",
      "order": {"_key": "asc"},
      "size": 200
    }
  }}
}
```

The output of this query will show each `_index` name, its first and last timestamp values. In the majority of cases the date on the index name is the date it was created. We expect to see that the `first_time` is _after_ the index's created date and the `last_time` is _before_ the created date of the next index:

```console
{
  "aggregations": {
    "2": {
      "buckets": [
        {
          "doc_count": 14,
   // The index date is 2022.01.26
          "key": ".ds-comments-2022.01.26-000007",       
   //  first_time is 2022.01.27, day after created date
          "first_time": {
            "value": 1643309325812,
            "value_as_string": "2022-01-27T18:48:45.812Z"
          },
    // last_time is 2022.02.25, same date as the next index
          "last_time": {
            "value": 1645791445246,
            "value_as_string": "2022-02-25T12:17:25.246Z"
          }
        },
        {
          "doc_count": 13,
          "first_time": {
            "value": 1646330753677,
            "value_as_string": "2022-03-03T18:05:53.677Z"
          },
          "key": ".ds-comments-2022.02.25-000008",
          "last_time": {
            "value": 1648142801719,
            "value_as_string": "2022-03-24T17:26:41.719Z"
          }
        }
      ],
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0
    }
  }
}
```

### D) Determine the buckets of future timestamps

```console
GET /_search?filter_path=aggregations
{
  "size": 0,
  "query": {
    "bool": {
      "must": [],
      "filter": [ { "range": { "@timestamp": { "gte": "now" }}}]
    }
  },
  "aggs": {
    "time_buckets": {
      "auto_date_histogram": {
        "field": "@timestamp",
        "buckets": 10, // Target number of buckets
        "format": "yyyy-MM-dd'T'HH:mm:ssZ" // You can change the desired format to only give the `yyyy-MM` if the ranges are big
      }
    }
  }
}
```


If any of the indices have a `last_date` that significantly overlaps with later indices, there were future timestamps being written into the index. Also, any dates future to the date the query was ran are obvious signs of future timestamps.

## Resolve

There are various actions that can be taken to resolve / mitigate the issue. Depending on your scenario, you might want to consider which of the below would be applicable for your situation:
- If the performance issue is mainly on the Security Analyzer app, please do point 1 (Feature flag) first, and check if it helps.
- If there was a clear mistake in the data (for instance, a time server not configured correctly), point 4 (correcting data) would make sense.
- Sometimes, future dates in the data is part of very specific scenarios, and neither point 2 or point 3 and 4 makes sense to do. 
- Furthermore, when you are managing several hundreds of Elastic built detection rules, point 2 (duplicating and changing the rule) becomes unfeasible. Note, our dev is working on [making Elastic rules configurable, without forfeiting future updates](https://github.com/elastic/kibana/issues/174168).
- In these last two scenarios, it might be needed to execute a more elaborate plan, see point 5, to split data into separate indices, grouped by timestamp.

### 1) Feature Flag

> 📣 This section is now covered+owned by Dev in [Set Rule Data Sources > Exclusion Options](https://www.elastic.co/docs/solutions/security/detect-and-alert/set-rule-data-sources#exclusion-options). We recommend reviewing current feature flags and options there instead. 

(Leaving historical as doc fallback:) Starting v8.12 via [kibana#172162](https://github.com/elastic/kibana/pull/172162), users can override [Advanced Setting](https://www.elastic.co/guide/en/kibana/current/advanced-options.html) `securitySolution:excludeColdAndFrozenTiersInAnalyzer` to mitigate performance impact for the Security Solution's [Event Analyzer feature](https://www.elastic.co/guide/en/security/current/visual-event-analyzer.html) when querying the Cold and Frozen Tiers. This setting must be overridden per [Kibana Space](https://www.elastic.co/guide/en/kibana/current/xpack-spaces.html) (and _only_ effects the Security Visual Event Analyzer not other features including Security Rules, Kibana, or Elasticsearch queries.  Only mitigates performance impact; would not solve potential data quality issues).

### 2) @timestamp override on detection rules advanced settings

The `timestamp_override` (with for instance `event.ingested`) and `Do not use @timestamp as a fallback timestamp field` [settings](https://www.elastic.co/guide/en/security/current/rules-ui-create.html#rule-ui-advanced-params) on rules can help mitigate this issue. While `@timestamp` is the only field that can be used for the optimisation on coordinating nodes described above, Elasticsearch can still optimise query execution once the query is sent to the individual shards for evaluation.

Setting a timestamp override and disabling use of `@timestamp` as a fallback completely removes the `@timestamp` time range filter from the query generated by the rule. When an override is set and fallback is disabled, the time range filter becomes `“timestamp override must be in this range”` instead of `“timestamp override must be in this range OR (timestamp override must not exist and @timestamp must be in range)”.`

> **_NOTE:_** please note that both these settings is needed to be set as described.

Without the second part of the OR clause, the query is simpler - there’s no more dependence on `@timestamp` so the query can execute much more efficiently even when there is a huge range of `@timestamp` values. The downside is that documents that don’t populate the timestamp override will always be excluded from the result set.

### 3) Prevent

We recommend resolving future dates as close to problem source as possible. In our above example "say a singe host has a bad time configuration with a future date", we would recommend resolving the host's bad time configuration. The following prevention options are offered to mitigate current/future upstream issues from cascading into your Elasticsearch cluster.

We've outlined two example [Ingest Pipeline](https://www.elastic.co/guide/en/elasticsearch/reference/current/ingest.html) resolutions. Please pick the one more applicable to your use case:
A) Overwrite `@timestamp` to ingest time (better performing)
B) Compare `@timestamp` vs `now` and overwrite it to lower value (requires more processing)

(Both examples demonstrate resolving previous timestamps for article integrity, but outlined recommendations also apply to future timestamps.)

#### A) Overwrite `@timestamp` to ingest time

Here is an ingest processor that will just overwrite `@timestamp` with now and save existing value into another field. (This example sequentially uses documentation references: [starting point](https://www.elastic.co/guide/en/elasticsearch/reference/current/set-processor.html), [conditional](https://www.elastic.co/guide/en/elasticsearch/reference/current/ingest.html#conditionally-run-processor), [simulate](https://www.elastic.co/guide/en/elasticsearch/reference/current/simulate-pipeline-api.html), [set](https://www.elastic.co/guide/en/elasticsearch/reference/current/set-processor.html), [rename](https://www.elastic.co/guide/en/elasticsearch/reference/current/rename-processor.html).)

```console
POST _ingest/pipeline/_simulate
{
  "pipeline": {
  "processors": [
    {
      "rename": {
        "field": "@timestamp",
        "target_field": "@timestampsave"
      }
      ,
      "set": {
        "field": "_source.@timestamp",
        "value": "{{_ingest.timestamp}}"
      }
    }
    ]},
  "docs": [
    {
      "_source": {
        "@timestamp": "2022-11-03T14:47:14.048557323Z"
      }
    }
  ]
}
```

#### B) Convert future dates to `now`

This ingest processor will set `@timestamp` to be the minimum value of current `@timestamp` OR `now`. This will consume more processing power than a simple overwrite discussed in (A). (This example sequentially uses documentation references: [painless and datetime](https://www.elastic.co/guide/en/elasticsearch/painless/master/painless-datetime.html), [how to get now](https://discuss.elastic.co/t/how-to-get-current-date-using-painless-langage/137820), [painless script in processor](https://www.elastic.co/guide/en/elasticsearch/painless/master/painless-ingest.html).)

```console
POST _ingest/pipeline/_simulate
{
  "pipeline": {
    "processors": [
      {
        "script": {
          "description": "",
          "lang": "painless",
          "source": """
          long nowepoch = new Date().getTime();
          ZonedDateTime dateinstring = ZonedDateTime.parse(ctx["@timestamp"]);
          long dateinepoch = dateinstring.toEpochMilli();
          ctx.nowepoch = nowepoch;
          ctx.dateinepoch = dateinepoch;
          if (nowepoch > dateinepoch) {
            ctx["@timestamp"] = dateinepoch;
          }
          else {
           ctx["@timestamp"] = nowepoch;
          }
          """
        }
      }
    ]
  },
  "docs": [
    {
      "_source": {
        "@timestamp": "2022-11-03T14:47:14.048557323Z"
      }
    }
  ]
}
```

### 4) Clean-up

There are only three ways to clean up future timestamps in past indices. Kindly see the Searchable Snapshot section below for applicability in carrying out clean-up.

1. [Reindex](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-reindex.html) the data into new indices
2. Run [Update by Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-update-by-query.html) while [applying an above Ingest Pipeline](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-update-by-query.html)
    - Update by query pointing [Data Streams](https://www.elastic.co/guide/en/elasticsearch/reference/current/data-streams.html) is not working. Should pointing towards Data Stream's backing indices.
3. [Remove unnecessary indices](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-delete-index.html) or run [Delete by Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-delete-by-query.html) against only the problematic data
   - example of delete by query

> **_NOTE:_** please also see complete example section below.

```console
#---
# create index with a couple of documents
# - delete the test index1 to make sure we are starting with a new index and no stale data.
#---
DELETE /index1
PUT /index1/_doc/1
{
  "@timestamp": "2023-06-01T00:00:00.00"
}

PUT /index1/_doc/2
{
  "@timestamp": "2023-07-01T00:00:00.00"
}

PUT /index1/_doc/3
{
  "@timestamp": "2023-08-01T00:00:00.00"
}

#---
# verify mapping is date/time
#---
GET /index1/_mapping

#---
# test with search first
# range query - https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-range-query.html
# - provided exampl
#---
GET /index1/_search
{
  "query": {
    "range": {
      "@timestamp": {
        "format": "strict_date_optional_time",
        "gt": "2023-06-30T00:00:00.000"
      }
    }
  }
}

GET /index1/_search
{
  "query": {
    "range": {
      "@timestamp": {
        "format": "strict_date_optional_time",
        "gt": "2023-06-30T00:00:00.000",
        "lt": "2023-07-30T00:00:00.000"
      }
    }
  }
}




#---
# run delete
#---
POST /index1/_delete_by_query
{
  "query": {
    "range": {
      "@timestamp": {
        "format": "strict_date_optional_time",
        "gt": "2023-06-30T00:00:00.000"
      }
    }
  }
}

#---
# verify documents have been removed
#---
GET /index1/_search
{
  "query": {
    "range": {
      "@timestamp": {
        "format": "strict_date_optional_time",
        "gt": "2023-06-30T00:00:00.000"
      }
    }
  }
}
```

#### Searchable Snapshots

[Searchable Snapshot](https://www.elastic.co/guide/en/elasticsearch/reference/current/searchable-snapshots.html) indices are read-only, so to apply fixes under (2) or (3), you may need to restore the backing index, make changes on it, and then send it back through [ILM's Searchable Snapshot](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-searchable-snapshot.html) action. 

Resolution generally follows the following outline per index; apply to use case. But not for Data Stream.
Kindly note, it can be done manually or scripted (but writing custom code is outside the scope of Elastic Support). For example `INDEX_NAME` you'll look to

1. [Read Index Settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-get-settings.html) extracting values under `.store.snapshot` details to use in (2).
    ```console
    GET INDEX_NAME/_settings?filter_path=*.settings.index.store.snapshot
    ```
2. Craft [Snapshot Restore](https://www.elastic.co/guide/en/elasticsearch/reference/current/restore-snapshot-api.html) replacing values in upper-case with response from (1). 
    ```console
    POST _snapshot/REPOSITORY_NAME/SNAPSHOT_NAME/_restore
    {
      "index_settings": {
        "index.lifecycle.name": null,
        "index.routing.allocation": {"include._tier_preference": "data_hot"},
        "index.lifecycle.indexing_complete": true
      },
      "indices": "INDEX_NAME",
      "rename_pattern": "(.+)",
      "rename_replacement": "NEW_$1"
    }
    ```
    (Note: apply-forward any modifications you do to `rename_replacement` to following steps.)
3. [Remove ILM Policy](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-remove-policy.html) (which clears the ILM cache on index).
    ```console
    POST NEW_INDEX_NAME/_ilm/remove
    ```
3. [Delete original Index](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-close.html) so searches stop reporting duplicate data. Performance issues should mitigate with this step. Data is still stored in `SNAPSHOT_NAME` from (2).
    ```console
    POST INDEX_NAME/_close
    ```
4. Apply the desired clean-up step outlined above.
5. Apply the existing or new [ILM policy](https://www.elastic.co/guide/en/elasticsearch/reference/current/set-up-lifecycle-policy.html) to index. If index formats `^.*-yyyy.MM.dd-\\d+` then use ` index.lifecycle.parse_origination_date` otherwise manually set `index.lifecycle.origination_date` value. Replace placeholders in upper-case.
    ```console
    PUT NEW_INDEX_NAME/_settings
    {
      "index.lifecycle.name": "MY_ILM_POLICY",
      "index.lifecycle.parse_origination_date": true
    }
    ``` 
6. **NOTE**:  If your ILM policy induces [ILM Force Merge](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-forcemerge.html), you'll want to avoid stacking up too many tasks at once which can have massive performance implications, including inducing [master overwhelm](https://www.elastic.co/guide/en/elasticsearch/reference/current/discovery-troubleshooting.html) and/or [hot spotting](https://www.elastic.co/guide/en/elasticsearch/reference/current/hotspotting.html).
7. Once [ILM Explain](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-explain-lifecycle.html) reports `NEW_INDEX_NAME` goes back into a Searchable Snapshot, the original index's snapshot can be deleted. 
    ```console
    DELETE _snapshot/REPOSITORY_NAME/SNAPSHOT_NAME
    ```

If target is Data Stream, follow the steps below:

1. [Read Index Settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-get-settings.html) extracting values under `.store.snapshot` details to use in (2).
    ```console
    GET INDEX_NAME/_settings?filter_path=*.settings.index.store.snapshot
    ```
2. Craft [Snapshot Restore](https://www.elastic.co/guide/en/elasticsearch/reference/current/restore-snapshot-api.html) replacing values in upper-case with response from (1). 
    ```console
    POST _snapshot/REPOSITORY_NAME/SNAPSHOT_NAME/_restore
    {
      "index_settings": {
        "index.lifecycle.name": null,
        "index.routing.allocation": {"include._tier_preference": "data_hot"},
        "index.lifecycle.indexing_complete": true
      },
      "indices": "INDEX_NAME",
      "rename_replacement": "NEW_INDEX_NAME"      # <- Do not put the Data Stream name
    }
    ```
3. Reindex to Data Stream while applying ingest pipeline from the desired clean-up step outlined above.

4. Remove "NEW_INDEX_NAME"

5. Keep in mind that reindexed data will be in the current writing index


### 5) Shrinking the timestamp range per index

If the use case would prevent you from updating the `@timestamp` fields, there is an alternative workaround that can be implemented to help alleviate most of the performance impact. The idea of this workaround would be to reindex future-timestamped data into their own index / indices without changing the data, so that there is a narrower timestamp range in these indices.

If there are only a small number of documents affected, they can all be reindexed to one index (per source). However, if there is a significant number of documents with this problem, group the documents with the future-timestamped data into separate indices, so that the time ranges do not overlap, and  provide a logical grouping. You will have to investigate your data to be able to determine how to group the data. 
- See `Search for future docs` from the detection section in the article to get a list of indices with the counts of future-timestamp documents.
- See `Determine the buckets of future timestamps` from the detection section in the article to determine how to split up the data.

After splitting up the indices (using, for instance `_reindex`), you can create different aliases so that you have control over which of these are used by which detection rule or query. Remember to also delete the reindexed documents from the original indices (delete by query).

The performance improvement with these split indices will give a big improvement, but you can also experiment leaving these in the hot or warm tier, if performance has not improved a lot.

