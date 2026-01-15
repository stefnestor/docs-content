---
mapped_pages:
  - https://www.elastic.co/guide/en/elasticsearch/reference/current/task-queue-backlog.html
applies_to:
  stack:
products:
  - id: elasticsearch
---

# Task queue backlog [task-queue-backlog]

% **Product:** Elasticsearch<br> **Deployment type:** Elastic Cloud % Enterprise, Elastic Cloud Hosted, Elastic Cloud on Kubernetes, Elastic
% Self-Managed <br> **Versions:** All

A backlogged task queue can lead to [rejected requests](/troubleshoot/elasticsearch/rejected-requests.md) or an [unhealthy cluster state](/troubleshoot/elasticsearch/red-yellow-cluster-status.md). Contributing factors can include [uneven or resource constrained hardware](/troubleshoot/elasticsearch/hotspotting.md#causes-hardware), a large number of tasks triggered at the same time, expensive tasks that are using [high CPU](/troubleshoot/elasticsearch/high-cpu-usage.md) or are inducing [high JVM](/troubleshoot/elasticsearch/high-jvm-memory-pressure.md), and long-running tasks.


## Diagnose a backlogged task queue [diagnose-task-queue-backlog] 

To identify the cause of the backlog, try these diagnostic actions.

* [Check thread pool status](#diagnose-task-queue-thread-pool)
* [Inspect node hot threads](#diagnose-task-queue-hot-thread)
* [Identify long-running node tasks](#diagnose-task-queue-long-running-node-tasks)
* [Look for long-running cluster tasks](#diagnose-task-queue-long-running-cluster-tasks)


### Check thread pool status [diagnose-task-queue-thread-pool] 

Use the [cat thread pool API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cat-thread-pool) to monitor active threads, queued tasks, rejections, and completed tasks:

```console
GET /_cat/thread_pool?v&s=t,n&h=type,name,node_name,active,queue,rejected,completed
```

There are a number of things that you can check as potential causes for the queue backlog:

* Look for continually high `queue` metrics, which indicate long-running tasks or [CPU-expensive tasks](high-cpu-usage.md).
* Look for bursts of elevated `queue` metrics, which indicate opportunities to spread traffic volume.
* Determine whether thread pool issues are specific to a [node role](/deploy-manage/distributed-architecture/clusters-nodes-shards/node-roles.md).
* Check whether a specific node is depleting faster than others within a [data tier](/manage-data/lifecycle/data-tiers.md). This might indicate [hot spotting](/troubleshoot/elasticsearch/hotspotting.md).


### Inspect node hot threads [diagnose-task-queue-hot-thread] 

If a particular thread pool queue is backed up, periodically poll the [nodes hot threads API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-nodes-hot-threads) to gauge the thread’s progression and ensure it has sufficient resources:

```console
GET /_nodes/hot_threads
```

Although the hot threads API response does not list the specific tasks running on a thread, it provides a summary of the thread’s activities. You can correlate a hot threads response with a [task management API response](https://www.elastic.co/docs/api/doc/elasticsearch/group/endpoint-tasks) to identify any overlap with specific tasks. For example, if the hot threads response indicates the thread is `performing a search query`, you can [check for long-running search tasks](#diagnose-task-queue-long-running-node-tasks) using the task management API.


### Identify long-running node tasks [diagnose-task-queue-long-running-node-tasks] 

Long-running tasks can also cause a backlog. Use the [task management API](https://www.elastic.co/docs/api/doc/elasticsearch/group/endpoint-tasks) to check for excessive `running_time_in_nanos` values:

```console
GET /_tasks?pretty=true&human=true&detailed=true
```

You can filter on a specific `action`, such as [bulk indexing](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-bulk) or search-related tasks. These tend to be long-running.

* Filter on [bulk index](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-bulk) actions:

    ```console
    GET /_tasks?human&detailed&actions=indices:*write*
    ```

* Filter on search actions:

    ```console
    GET /_tasks?human&detailed&actions=indices:*search*
    ```


Long-running tasks might need to be [canceled](#resolve-task-queue-backlog-stuck-tasks).

Refer to [this video](https://www.youtube.com/watch?v=lzw6Wla92NY) for a walkthrough of how to troubleshoot the [task management API](https://www.elastic.co/docs/api/doc/elasticsearch/group/endpoint-tasks) output. 

You can also check the [Tune for search speed](/deploy-manage/production-guidance/optimize-performance/search-speed.md) and [Tune for indexing speed](/deploy-manage/production-guidance/optimize-performance/indexing-speed.md) pages for more information.

### Look for long-running cluster tasks [diagnose-task-queue-long-running-cluster-tasks] 

Use the [cat pending tasks API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cat-pending-tasks) to identify delays in cluster state synchronization:

```console
GET /_cat/pending_tasks?v=true
```

Cluster state synchronization can be expected to fall behind when a [cluster is unstable](/troubleshoot/elasticsearch/troubleshooting-unstable-cluster.md), but otherwise this state usually indicates an unworkable [cluster setting](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cluster-get-settings) override or traffic pattern.

There are a few common `source` issues to check for:

* `ilm-`: [{{ilm}} ({{ilm-init}})](/manage-data/lifecycle/index-lifecycle-management.md) polls every `10m` by default, as determined by the [`indices.lifecycle.poll_interval`](elasticsearch://reference/elasticsearch/configuration-reference/index-lifecycle-management-settings.md) setting. This starts asynchronous tasks executed by the node tasks. If {{ilm-init}} continually reports as a cluster pending task, this setting likely is being overridden. Otherwise, the cluster likely has misconfigured [indices count relative to master heap size](/deploy-manage/production-guidance/optimize-performance/size-shards.md#shard-count-recommendation).
* `put-mapping`: {{es}} enables [dynamic mapping](/manage-data/data-store/mapping/dynamic-mapping.md) by default. This, or the [update mapping API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-indices-put-mapping), triggers a mapping update. In this case, the corresponding cluster log will contain an `update_mapping` entry with the name of the affected index.
* `shard-started`: Indicates [active shard recoveries](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cat-recovery). Overriding [`cluster.routing.allocation.*` settings](elasticsearch://reference/elasticsearch/configuration-reference/cluster-level-shard-allocation-routing-settings.md#cluster-shard-allocation-settings) can cause pending tasks and recoveries to back up.


## Recommendations [resolve-task-queue-backlog] 

After identifying problematic threads and tasks, resolve the issue by increasing resources or canceling tasks.

### Address CPU-intensive tasks [resolve-task-queue-backlog-cpu]

If an individual task is causing a [thread pool `queue`](#diagnose-task-queue-thread-pool) due to [high CPU usage](high-cpu-usage.md), try [cancelling the task](#resolve-task-queue-backlog-stuck-tasks) and then optimizing it before retrying.

This problem can surface due to a number of possible causes:

* Creating new  tasks or modifying scheduled tasks which either run frequently or are broad in their effect, such as [{{ilm}}](/manage-data/lifecycle/index-lifecycle-management.md) policies or [rules](/explore-analyze/alerts-cases.md)
* Performing traffic load testing
* Doing extended look-backs, especially across [data tiers](/manage-data/lifecycle/data-tiers.md)
* [Searching](https://www.elastic.co/docs/api/doc/elasticsearch/group/endpoint-search) or performing [bulk updates](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-bulk) to a high number of indices in a single request


### Cancel stuck tasks [resolve-task-queue-backlog-stuck-tasks] 

If an active task’s [hot thread](#diagnose-task-queue-hot-thread) shows no progress, consider [canceling the task](https://www.elastic.co/docs/api/doc/elasticsearch/group/endpoint-tasks#task-cancellation) if it's flagged as `cancellable`.


### Address hot spotting [resolve-task-queue-backlog-hotspotting] 

If a specific node’s thread pool is depleting faster than its [data tier](/manage-data/lifecycle/data-tiers.md) peers, try addressing uneven node resource utilization, also known as "hot spotting". For details about reparative actions you can take, such as rebalancing shards, refer to the [Hot spotting](hotspotting.md) troubleshooting documentation.

### Increase available resources [resolve-task-queue-backlog-resources] 

By default, {{es}} allocates processors equal to the number reported available by the operating system. You can override this behaviour by adjusting the value of [`node.processors`](elasticsearch://reference/elasticsearch/configuration-reference/thread-pool-settings.md#node.processors), but this advanced setting should be configured only after you've performed load testing.

In some cases, you might need to increase the problematic thread pool `size`. For example, it might help to increase a stuck [`force_merge` thread pool](elasticsearch://reference/elasticsearch/configuration-reference/thread-pool-settings.md). If the setting is automatically calculated to `1` based on available CPU processors, then increasing the value to `2` is indicated in `elasticsearch.yml`, for example:

```yaml
thread_pool.force_merge.size: 2
```
