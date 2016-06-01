# Overview

This is a mini tutorial explaining how to store metrics in Xenon (based Lucene), mainly designed for those 
who intend to create and handle large amount of services using Xenon framework.

There are existing open source and proprietary databases for storing metrics and specifically time series metrics such as: ElasticSearch(based on Lucene), InfluxDB etc.

There are many use cases for storing metrics, for example storing performance metrics such as cpu, memory, network. These metrics are mainly used for monitoring the system health in various kind of loads.
Storing metrics usually require handling a quite decent amount of data streamed in a high volume. 10,000 to 50,000 data points per second aren't something unusual. The collected data is mostly relevant on high loads so we need to be sure that storing and searching will work at critical time with high load.

Considering all points specified above, we'll address some topics such as optimizing metric structure, efficient storing, and general tweaking aspects, in order to meet the conditions that the system should support.

# Metric Structure

Storing 10,000 metrics per second must be fast enough otherwise the server might become unresponsive.
One aspect we should consider is the metric object size, which might have a significant affect on performance.
Xenon documents shouldn't be too big otherwise it'll have impact indexing performance. 

On the other hand Xenon actually adds about 300 bytes of data to the each created service, so it might be very costly to create a tiny service. (Every Xenon objects includes several fields like URI, version, timestamp, etc which have more overhead compared to the service fields).

# Metric Service

An important thing we should pay attention to when dealing with high metric creation throughput is that the service preferably shouldn't do anything that requires computing or i/o intensive operations, the reason is that it impacts throughput performance.

In case a service having replication option (**ServiceOption.REPLICATION**) enabled, then every service state update will be replicated among all peer nodes, in case the node group is comprised of several peer nodes the replication might take additional time for managing the replication process and we might observe throughput degradation on multi node replication service.
 
To improve replication overhead we can consider using **DEFAULT_3X_NODE_SELECTOR** service option which is a limited replication node selector that replicates state updates to 3 selected nodes per self link.
Find more about 3x replication: **https://github.com/vmware/xenon/wiki/NodeSelectorService**

Moreover metric service can become quite large (1B services), Xenon persistent services are being cached on node load and on service creation which means a large amount of services will be loaded in memory. To prevent caching of those services we can use **ON_DEMAND_LOAD** service option that will eventually cause idle services to stop shortly after they created and to be removed from cache.

Another significant point, if you'd like to fetch stored metrics with a query then the query will fetch the services directly from Lucene index and thus they won't be loaded into cache, that means fetching services with Xenon query won't affect memory load.

**WIP** **WIP** **WIP**