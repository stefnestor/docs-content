---
navigation_title: "Overview"
---

# Monitoring overview [monitoring-overview]


When you monitor a cluster, you collect data from the {{es}} nodes, {{ls}} nodes, {{kib}} instances, APM Server, and Beats in your cluster. You can also collect logs.

All of the monitoring metrics are stored in {{es}}, which enables you to easily visualize the data in {{kib}}. By default, the monitoring metrics are stored in local indices.

::::{admonition}
If you’re using Elastic Cloud Hosted, then you can use AutoOps to monitor your cluster. AutoOps significantly simplifies cluster management with performance recommendations, resource utilization visibility, real-time issue detection and resolution paths. For more information, refer to [Monitor with AutoOps](/deploy-manage/monitor/autoops.md).

::::


::::{tip}
In production, we strongly recommend using a separate monitoring cluster. Using a separate monitoring cluster prevents production cluster outages from impacting your ability to access your monitoring data. It also prevents monitoring activities from impacting the performance of your production cluster. For the same reason, we also recommend using a separate {{kib}} instance for viewing the monitoring data.
::::


You can use {{agent}} or {{metricbeat}} to collect and ship data directly to your monitoring cluster rather than routing it through your production cluster.

The following diagram illustrates a typical monitoring architecture with separate production and monitoring clusters. This example shows {{metricbeat}}, but you can use {{agent}} instead.

:::{image} ../../../images/elasticsearch-reference-architecture.png
:alt: A typical monitoring environment
:::

If you have the appropriate license, you can route data from multiple production clusters to a single monitoring cluster. For more information about the differences between various subscription levels, see: [https://www.elastic.co/subscriptions](https://www.elastic.co/subscriptions)

::::{important}
In general, the monitoring cluster and the clusters being monitored should be running the same version of the stack. A monitoring cluster cannot monitor production clusters running newer versions of the stack. If necessary, the monitoring cluster can monitor production clusters running the latest release of the previous major version.
::::


