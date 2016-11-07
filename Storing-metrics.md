# Overview

Sometimes you need to store time-series data such as metrics in Xenon. This kind of data has some unique charectaristics:
* You often need to store a lot of metrics (over time)
* Each metric data is typically very small
* Most of the metrics are never or seldom read directly - a small portion of them need to be fetched through queries
* The data is not considered critical - losing some of it in case of a catastrophic failure is typically considered OK
* Metrics are often aggregated so that the size of the raw data can be kept reasonable

This page contains a few best practices for modeling metrics in Xenon.

# Use a factory to represent metrics per collection source or type

Each metric / stat / log item can be viewed as a document, that is created through a POST to a factory.

For example consider a factory listening on

```
/metrics/firewall-events
```

and then a client can add a new item using a POST:

```
POST /metrics/firewall-events
{"documentSelfLink":"item0","sourceUri":"http://somehost.com:8000","eventTimeMillis":"1466621098568",...}

```

# Service Options

The service representing the metric item should be marked with:
 * ServiceOption.PERSISTANCE
 * ServiceOption.REPLICATION
 * ServiceOption.ON_DEMAND_LOAD
 * ServiceOption.IMMUTABLE

## On demand load
StatefulServices are usually started by a factory shortly after the Xenon host starts. In the case of metrics it makes much more sense to load them only when a client wants to access them. By toggling on ServiceOption.ON_DEMAND_LOAD on the StatefulService representing a metric you tell Xenon to load the metric only on-demand.

## Immutable

The immutable option enables several key optimizations during indexing, replication and queries. Immutable services assume a single version so by pass the index check for previous version on the original POST.

## Replication

Metric data might require its own node group, due to the increased disk I/O and ingestion rates. If metric data is stored in the same node group as other xenon service types (tasks, desired state, configuration) consider using
the limited replication selectors (3x) to only replicate up to N nodes, even a node group with K >> N nodes. If you enable limited replicated, you will need to use the **broadcast** query option to aggregate results

# Query Options

When querying over metric or immutable data yu can avoid the latest version filtering that occurs for multi version documents, speeding up query result processing. Specify:

* QueryOption.INCLUDE_ALL_VERSIONS

