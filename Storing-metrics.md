# Overview

Sometimes you need to store time-series data such as metrics in Xenon. This kind of data has some unique charectaristics:
* You often need to store a lot of metrics (over time)
* Each metric data is typically very small
* Most of the metrics are never or seldom read - a small portion of them need to be fetched on-demand for some analysis
* The data is not considered critical - losing some of it in case of a catastrophic failure is typically considered OK
* Metrics are often aggregated so that the size of the raw data can be kept reasonable

This page contains a few best practices for modeling metrics in Xenon.

# Toggle ServiceOption.ON_DEMAND_LOAD

StatefulServices are usually started by a factory shortly after the Xenon host starts. In the case of metrics it makes much more sense to load them only when a client wants to access them. By toggling on ServiceOption.ON_DEMAND_LOAD on the StatefulService representing a metric you tell Xenon to load the metric only on-demand.

# Metric Service

An important thing we should pay attention to when dealing with high metric creation throughput is that the service preferably shouldn't do anything that requires computing or i/o intensive operations, the reason is that it impacts throughput performance.

In case a service having replication option (**ServiceOption.REPLICATION**) enabled, then every service state update will be replicated among all peer nodes, in case the node group is comprised of several peer nodes the replication might take additional time for managing the replication process and we might observe throughput degradation on multi node replication service.
 
To improve replication overhead we can consider using **DEFAULT_3X_NODE_SELECTOR** service option which is a limited replication node selector that replicates state updates to 3 selected nodes per self link.
Find more about 3x replication: **https://github.com/vmware/xenon/wiki/NodeSelectorService**

Moreover metric service can become quite large (1B services), Xenon persistent services are being cached on node load and on service creation which means a large amount of services will be loaded in memory. To prevent caching of those services we can use **ON_DEMAND_LOAD** service option that will eventually cause idle services to stop shortly after they created and to be removed from cache.

Another significant point, if you'd like to fetch stored metrics with a query then the query will fetch the services directly from Lucene index and thus they won't be loaded into cache, that means fetching services with Xenon query won't affect memory load.

**WIP** **WIP** **WIP**