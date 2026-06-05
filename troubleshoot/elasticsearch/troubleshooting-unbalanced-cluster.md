---
navigation_title: Unbalanced clusters
mapped_pages:
  - https://www.elastic.co/guide/en/elasticsearch/reference/current/troubleshooting-unbalanced-cluster.html
applies_to:
  stack:
products:
  - id: elasticsearch
---

# Troubleshoot an unbalanced cluster [troubleshooting-unbalanced-cluster]

:::{include} /deploy-manage/_snippets/autoops-callout-with-ech.md
:::

{{es}} assumes all nodes within a [data tier](/manage-data/lifecycle/data-tiers.md) share the same hardware profile. This enables its Allocation feature to balance distributing index shards across target nodes.

Allocation sequentially computes from [cluster-level settings and filters](elasticsearch://reference/elasticsearch/configuration-reference/cluster-level-shard-allocation-routing-settings.md), then an individual node's [disk watermark](/troubleshoot/elasticsearch/fix-watermark-errors.md), and finally from shard awareness. Within these logical prerequisites, such as [data tiers](/manage-data/lifecycle/data-tiers.md), {{es}} [balances shards](elasticsearch://reference/elasticsearch/configuration-reference/cluster-level-shard-allocation-routing-settings.md) to achieve a good compromise between:

* current shard count
* forecasted disk usage
* write load (for indices in data streams)

There is no guarantee that individual components will be evenly spread across the nodes. This could happen if some nodes have fewer shards, or are using less disk space, but are assigned shards with higher write loads.

When rebalancing shards, {{es}} does not consider the amount or complexity of search queries. This is indirectly achieved by balancing shard count and disk usage. It also does not consider:

* current [CPU usage](/troubleshoot/elasticsearch/high-cpu-usage.md)
* current [JVM memory pressure](/troubleshoot/elasticsearch/high-jvm-memory-pressure.md)
* [task queue backlog](/troubleshoot/elasticsearch/task-queue-backlog.md)
* which nodes [coordinate the related tasks](/deploy-manage/distributed-architecture/reading-and-writing-documents.md)
* which node is elected-master
* write load of aliases or standalone indices

## Check balance [troubleshooting-unbalanced-cluster-check]

To check the cluster's balance, use the [cat allocation command]({{es-apis}}operation/operation-cat-allocation) to list workloads per node:

```console
GET /_cat/allocation?v
```

The API returns the following response:

```text
shards shards.undesired write_load.forecast disk.indices.forecast disk.indices disk.used disk.avail disk.total disk.percent host          ip            node      node.role
    35                0  3.8590562438622698               744.1gb      496.1gb   523.2gb      1.6tb      2.1tb           23 10.224.62.48  10.224.62.48  hot-09    hirs
    47                0   4.020483253384615                 407gb      237.2gb   256.2gb      1.8tb      2.1tb           11 10.224.62.92  10.224.62.92  hot-09    hirs
    63                1                 0.0                 2.6tb        1.8tb     1.8tb     10.1tb     11.9tb           15 10.224.62.119 10.224.62.119 cold-07   c
    64                3                 0.0                 2.6tb        1.7tb     1.8tb     10.1tb     11.9tb           15 10.224.141.89 10.224.141.89 cold-05   c
```

This response contains the following information that influences balancing:

* `shards` is the current number of shards allocated to the node
* `shards.undesired` is the number of shards that needs to be moved to other nodes to finish balancing
* `disk.indices.forecast` is the expected disk usage according to projected shard growth
* `write_load.forecast` is the projected total write load associated with this node

A cluster is considered balanced when all shards are in their desired locations, which means that no further shard movements are planned (all `shards.undesired` values are equal to 0).

## Balancing expectations [troubleshooting-unbalanced-cluster-check]

Some operations such as node restarting, decommissioning, or changing cluster allocation settings are disruptive and might require multiple shards to move in order to rebalance the cluster.

Shard movement order is not deterministic and mostly determined by the source and target node readiness to move a shard. While rebalancing is in progress some nodes might appear busier than others.

When a shard is allocated to an undesired node it uses the resources of the current node instead of the target. This might cause a [hotspot](/troubleshoot/elasticsearch/hotspotting.md) (disk or CPU) when multiple shards reside on the current node that have not been moved to their corresponding targets yet.

You can monitor shard migrations using the [cat recovery command]({{es-apis}}operation/operation-cat-recovery), along with their migrated `bp` bytes percent of `tb` total bytes:

```console
GET _cat/recovery?v=true&expand_wildcards=all&active_only=true&h=time,tb,bp,top,ty,st,snode,tnode,idx,sh&s=time:desc
```

## Divergent balances [troubleshooting-unbalanced-cluster-check]

If a cluster takes a long time to finish rebalancing you might find the following log entries:

```text
[WARN][o.e.c.r.a.a.DesiredBalanceReconciler] [10%] of assigned shards (10/100) are not on their desired nodes, which exceeds the warn threshold of [10%]
```

This is not concerning as long as the number of such shards is decreasing and this warning appears occasionally, for example after rolling restarts or changing allocation settings.

If the cluster has this warning repeatedly for an extended period of time (multiple hours), it is possible that the desired balance is diverging too far from the current state.

If so, you should:

1. Increase the [`cluster.routing.allocation.balance.threshold`](elasticsearch://reference/elasticsearch/configuration-reference/cluster-level-shard-allocation-routing-settings.md#shards-rebalancing-heuristics) setting to reduce the sensitivity of the algorithm that tries to level up the shard count and disk usage within the cluster.

1. $$$delete-desired-balance-request-example$$$ Reset the desired balance using the following API call:

    ```console
    DELETE /_internal/desired_balance
    ```

    ::::{note}
    If your deployment runs on an orchestrating platform such as {{ech}}, {{ece}}, or {{eck}}, the desired balance can only be reset by a user with operator privileges. Refer to [operator privileges](/deploy-manage/users-roles/cluster-or-deployment-auth/operator-privileges.md) for more information.
    ::::
