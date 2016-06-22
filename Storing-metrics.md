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

# Disable replication

Metrics are often not considered critical data so the overhead of replicating them among hosts is often not worth the data protection gain. It is therefore recommended to not replicate metrics. In your metrics queries you need to use QueryOption.BROADCAST to collect the metrics from the various hosts in the node group. Typically you would query for a small subset of the metrics, e.g. ones created within a specific time interval.
