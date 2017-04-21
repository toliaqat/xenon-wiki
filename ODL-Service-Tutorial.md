# On-Demand-Load Services

A service author can toggle ServiceOption.ON_DEMAND_LOAD to let the runtime know that this service
instance should not be restarted on host restart, and, it should be automatically stopped when its idle.

Use cases:
Enable this option if the service fits this criteria:
1. It represents a metric, log item, stat etc. Something with a single version and immutable. 
That service will be indexed, but there is no need to stay resident, or restarted on node restart.
The queries will not cause it to load, since they go to the index. If you expect many millions of such services,
definately mark as ON_DEMAND_LOAD.
Note that if you model metrics using the same fixed link,
but millions of discrete versions, on-demand load is not as useful,
since you dont have that many service instances. So don't set it in that case.
See the recommendations here: [Storing metrics](Storing-metrics)
2. Task services. Once a task is done, no need for it to stay resident.
And no need to restart it on node restart. You can still query across all tasks,
again without causing a reload. And, if you ever access directly a specific document/task, xenon will load it

## Synchronization for ODL Services
For ODL service the synchronization logic kick’s in only when the service is accessed. In the case of host restart, Xenon will
synchronize the ODL services in this restarted host and could have stale data. The implementors need to make sure
that they kick in the synchronization by themselves on host restart.
