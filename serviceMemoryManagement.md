# Overview

One of the key xenon requirement is to not be memory limited. This means that you can have more services / documents than available heap, per host. The runtime achieves this through two independent mechanisms:
 * Memory pressure and grooming - Every service host periodically computes available memory, using hints provided by the user (the memory threshold setter methods on ServiceHost), and pauses services to disk. Service state is already kept on the index, but the service itself has a small foot print (~400 bytes) so we eliminate that as well by pausing the service context (see [service pause / resume](./servicePauseResume)
 * On demand load - A service author can toggle **ServiceOption.ON_DEMAND_LOAD** to let the runtime know that this service instance should not be restarted on host restart, and, it should be automatically stopped when its idle.

## Service Pause / Resume
Please refer to the [pause / resume](./servicePauseResume) page for details

## On demand load
The **ServiceOption.ON_DEMAND_LOAD** option should be used for the following service patterns:
 1 The service represents a metric, log item, stat. Something with a single version, immutable. That service will be indexed, but there is no need to stay resident, or restarted on node restart. The queries will not cause it to load, since they go to the index :slightly_smiling_face:. If you expect many millions of such services, definitely mark as ON_DEMAND_LOAD. Note that if you model metrics using the same fixed link, but millions of discrete versions, on-demand load is not as useful, since you dont have that many service instances. See the recommendations here in the [metrics page](./Storing-metrics)
 1 Task services. Once a task is done, no need for it to stay resident. And no need to restart it on node restart. You can still query across all tasks, again without causing a reload. And, if you ever access directly a specific document/task, xenon will load it (edited)

