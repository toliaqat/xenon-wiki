# Overview

Sometimes you need to store time-series data such as metrics in Xenon. This kind of data has some unique charectaristics:
* You often need to store a lot of metrics (over time)
* Each metric data is typically very small
* Most of the metrics are never or seldom read - a small portion of them need to be fetched on-demand for some analysis
* The data is not considered critical - losing some of it in case of a catastrophic failure is typically considered OK
* Metrics are often aggregated so that the size of the raw data can be kept reasonable

This page contains a few best practices for modeling metrics in Xenon.

# Use a factory to represent metrics per collection source or type

Each metric / stat / log item can be viewed as a document, that is created through a POST to a factory. 

For example consider a factory listening on
`
/metrics/firewall-events
`

and then a client can add a new item using a POST. 
`
POST /metrics/firewall-events
{"documentSelfLink":"item0","sourceUri":"http://somehost.com:8000","eventTimeMillis":"1466621098568",...}

`
## Using PATCH on the same link instead of POST

Xenon has significantly better throughput for updates on a already created document, vs creating it for the first time. This is because we can avoid some index existence checks. For example on the same system, with a SSD drive, we can do 300,000 indexed PUTs / sec, vs 60,000 POSTs / sec.

So, to model the same metrics as above, you can simple use the same self link, but keep doing PUTs with new metrics. When querying across metrics, **include QueryOption.INCLUDE_ALL_VERSIONS**

Example:
 * Create the fixed link once, using the fixed hint "event-stream"
`
POST /metrics/firewall-events
{"documentSelfLink":"event-stream","sourceUri":"http://somehost.com:8000","eventTimeMillis":"1466621088568",...}
`

 * Append the next metric, as a new version, by using put
`
PUT /metrics/firewall-events/event-stream
{"sourceUri":"http://newhost.com:8000","eventTimeMillis":"1466621098568",...}
`
# Recommended Service Options 

The service representing the metric item should be marked with:
 * ServiceOption.PERSISTANCE
 * ServiceOption.ON_DEMAND_LOAD

## On demand load
StatefulServices are usually started by a factory shortly after the Xenon host starts. In the case of metrics it makes much more sense to load them only when a client wants to access them. By toggling on ServiceOption.ON_DEMAND_LOAD on the StatefulService representing a metric you tell Xenon to load the metric only on-demand.

## Disable eager replication
Metrics are often not considered critical data so the overhead of replicating them among hosts is often not worth the data protection gain, at least with eager replication. It is therefore recommended to not replicate metrics. In your metrics queries you need to use QueryOption.BROADCAST to collect the metrics from the various hosts in the node group. Typically you would query for a small subset of the metrics, e.g. ones created within a specific time interval.

## Backup of metric data

The runtime supports back-up of the entire index through a PATCH request to /core/management. A service can issue a backup request per host, periodically (once an hour, day, etc)

# Limit retention of raw data
The amount of metrics collected over time can be huge, hence it is recommended to use an aggregation & expiration approach: set the documentExpirationTime on stored metrics (e.g. for up to 1 day) an aggregate in units (e.g. hourly, daily, weekly, etc.)

# Stop metrics right after creation
Xenon cache will stop ON_DEMAND_LOAD services after a period of inactivity in case of memory pressure, however if your incoming metrics rate is particularly high you shouldn't wait for this mechanism to kick-in - consider stopping a metric service right after its creation. That will remove its memory footprint from the host immediately.

## Service self stop
To self stop a service issue a DELETE operation, but add a request header indicating the service should just stop, not marked deleted in the index:

```

 Operation.createDelete(s.getUri())
        .addPragmaDirective(Operation.PRAGMA_DIRECTIVE_NO_INDEX_UPDATE)
        .setReplicationDisabled(true).sendWith(this);

```
