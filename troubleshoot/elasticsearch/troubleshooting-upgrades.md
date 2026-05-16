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

Usually, [{{es}} upgrades](/deploy-manage/upgrade/deployment-or-cluster/elasticsearch.md) proceed smoothly due to due diligence in upgrade [planning](/deploy-manage/upgrade/plan-upgrade.md) and [preparation](/deploy-manage/upgrade/prepare-to-upgrade.md).

We have compiled the most common errors encountered throughout the upgrade process. Some are included below while others

* [Troubleshoot Upgrade Assistant](troubleshoot/elasticsearch/troubleshooting-upgrade-assistant.md)

This guide outlines {{es}} logs which indicate either upgrade blocking issues or fatal node start-up errors.



## 

{{es}} supports running two versions during a rolling upgrade, from an earlier version to later version. It does not ever support running more than two versions. It does not support two versions beyond the duration of the rolling upgrade. To monitor which nodes have been upgraded, use the [CAT nodes](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cat-nodes) API:

```console
GET _cat/nodes?v=true&h=name,ip,version,uptime&s=uptime
```

A restarted node must sequentially

* pass [bootstrap checks](/deploy-manage/deploy/self-managed/bootstrap-checks.md#checks)
* [join cluster](/troubleshoot/elasticsearch/troubleshooting-unstable-cluster.md) as confirmed by `master node changed` log
* [allocate shards](/troubleshoot/elasticsearch/diagnose-unassigned-shards.md) if [`data` node](/deploy-manage/distributed-architecture/clusters-nodes-shards/node-roles.md#data-node-role)

### bootstrap

- https://github.com/elastic/kibana/issues/266733
https://support.elastic.dev/knowledge/view/0defaeb6
https://support.elastic.dev/knowledge/view/ce40bf87
```text
fatal exception while booting Elasticsearch java.lang.IllegalStateException: The index [XXXXX] created in version [7.x.x] with current compatibility version [7.x.x] must be marked as read-only using the setting [index.blocks.write] set to [true] before upgrading to 9.x.x.
```
- https://discuss.elastic.co/t/fatal-exception-while-booting-elasticsearch/322390
```text
AL1-0131","elasticsearch.cluster.name":"elasticsearch","error.type":"java.nio.file.AccessDeniedException","error.message":"/etc/elasticsearch/tmp/empty8245557451022466264tmp","error.stack_trace":"java.nio.file.AccessDeniedException: /etc/elasticsearch/tmp/empty8245557451022466264tmp\n\tat java.base/

java.lang.ExceptionInInitializerError: null
Caused by: java.lang.UnsupportedOperationException: Failed to allocate closure
```text
https://www.elastic.co/docs/deploy-manage/deploy/self-managed/executable-jna-tmpdir
- https://support.elastic.dev/knowledge/view/f6d8cbf1 
https://support.elastic.dev/knowledge/view/961cbc48
```
fatal exception while booting Elasticsearch
java.lang.IllegalArgumentException: unknown setting [X] please check that any required plugins are installed, or check the breaking changes documentation for removed settings
```
- (?) initial master nodes
- https://support.elastic.dev/knowledge/view/967ed30c
```text
ERROR: fatal exception while booting Elasticsearch org.elasticsearch.ElasticsearchException: failed to load metadata ...
Caused by: org.elasticsearch.gateway.CorruptStateException: java.lang.IllegalArgumentException: Unexpected field [transport_version]
```
- https://support.elastic.dev/knowledge/view/557e7751
```text
[instance-0000000040] fatal exception while booting Elasticsearch
java.lang.IllegalStateException: Cannot start this node because it holds metadata for indices with version [6.5.4] with which this node of version [8.18.3] is incompatible. Revert this node to version [7.17.9] and delete any indices with versions earlier than [7.0.0] before upgrading to version [8.18.3]. If all such indices have already been deleted, revert this node to version [7.17.9] and wait for it to join the cluster to clean up any older indices from its metadata.
```




## Rolling upgrades considerations [upgrade-issues]

During a rolling upgrade, the cluster continues to operate normally. New functionality is either inactive or operates in a backward-compatible mode until the last node of earlier version leaves the cluster. New functionality becomes operational when all nodes in the cluster are running the later version. 

Usually, the earlier version nodes only leave the cluster when you shut them down to upgrade them. In this case, the last earlier version node leaves the cluster when there are no more nodes to upgrade. 

### Premature exit

However, it is possible that an earlier version node might temporarily or permanently (until intervened) leave the cluster before you purposely shut it down due to [cluster fault detection](/deploy-manage/distributed-architecture/discovery-cluster-formation/cluster-fault-detection.md).

If all the remaining earlier version nodes unexpectedly leave the cluster during an upgrade, the cluster will consider itself to be fully-upgraded, automatically activate new functionality, and leave its backward-compatible mode.

Once that has happened, there is no way to return the cluster to a state that is compatible with the earlier version nodes. Nodes running the earlier version will not be able to join this fully-upgraded cluster. Their {{es}} logs will report `failed to join` issues due to `Caused by` errors like

* `node version [x.x.x] may not join a cluster comprising only nodes of version [y.y.y] or greater`
* `node with version [x.x.x] may not join a cluster with minimum version [y.y.y]`
* `node with system index mappings versions [y.y.y] may not join a cluster with minimum system index mappings versions [x.x.x]`
* `handshake with [NODE_ID] failed: remote node version [x.x.x] is incompatible with local node version [y.y.y]`

{{es}} maintains the data in the data paths of the older nodes and will recover the cluster to health using this data after the nodes are fully upgraded. Therefore, to bring these nodes back into the cluster, upgrade them.

:::{note} :applies_to: { ece:, ess: }
Usually you can "Reapply" your latest [Deployment activity](/deploy-manage/deploy/elastic-cloud/keep-track-of-deployment-activity.md) {{es}} upgrade to finish upgrading. If the node out of cluster causes a [Cluster health](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cluster-health) status of `red`, then plans will be blocked for data safety. If this is the case, kindly [contact us](/troubleshoot/index.md#contact-us) with {{ech}} deployment ID or [{{ece}} diagnostic](/troubleshoot/deployments/cloud-enterprise/run-ece-diagnostics-tool.md) flagged `--deployments` for the problematic deployment ID.
:::


### stopping master-eligible nodes 

If you stop half or more of the master-eligible nodes all at once during the upgrade, the cluster will [become unavailable](/troubleshoot/elasticsearch/discovery-troubleshooting.md#discovery-no-master) due to insufficient [voting configurations](/deploy-manage/distributed-architecture/discovery-cluster-formation/modules-discovery-voting.md).

You must restart all the stopped master-eligible nodes to allow the cluster to re-form. If the re-formed cluster comprises only upgraded nodes, then the cluster will consider itself to be fully-upgraded, automatically activate new functionality, and leave its backward-compatible mode. In this case, upgrade all other nodes running the old version to enable them to join the re-formed cluster. [Upgrade the master-eligible nodes last](/deploy-manage/upgrade/deployment-or-cluster/elasticsearch.md#upgrade-order) to make it less likely that this occurs.

#### insufficient master-eligible nodes

Production environments should have [at least three master-eligible nodes for high availability](/deploy-manage/deploy/elastic-cloud/elastic-cloud-hosted-planning.md). In a testing or development environment with only one or two master-eligible nodes, you cannot avoid stopping half or more of the master-eligible nodes, so the cluster will always [become unavailable](/troubleshoot/elasticsearch/discovery-troubleshooting.md#discovery-no-master) at some point during the upgrade. When you restart the master-eligible nodes after this unavailability, the cluster will re-form with a single upgraded node, which is therefore fully-upgraded and will reject older nodes' attempts to re-join the cluster. Upgrade the master-eligible nodes last to avoid these rejections.

## allocation issues unique to upgrade

generic https://www.elastic.co/docs/troubleshoot/elasticsearch/cluster-allocation-api-examples 

- Incompatible Index Version
```text
illegal_argument_exception: The index [my_index] was created with version [X.X.X] but the minimum compatible version is [Y.Y.Y]

java.lang.IllegalStateException: index [my_index] version not supported: X.X.X maximum compatible index version is: Y.Y.Y

cannot upgrade node because incompatible indices created with version [X.X.X] exist, while the minimum compatible index version is [Y.Y.Y]. Upgrade your older indices by reindexing them in version [Z.Z.Z] first.
```
- Shard Allocation "Version Mismatch"
```text
cannot allocate replica shard to a node with version [X.X.X] since this is older than the primary version [Y.Y.Y]
```



## Post-upgrade

- 
Re-enable shard allocation: https://www.elastic.co/docs/deploy-manage/upgrade/deployment-or-cluster/elasticsearch#re-enable-shard-allocation 
Kibana server not ready yet: https://www.elastic.co/docs/troubleshoot/kibana/error-server-not-ready 
- transform upgrade mode 
https://support.elastic.dev/knowledge/view/bbb917e2
```text
Failed to bootstrap prebuilt rules and a status_exception: Cannot stop any Transform while the Transform feature is upgrading (408)
```

