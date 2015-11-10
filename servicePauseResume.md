# Overview

A DCP service instance consumes a small amount of memory to keep track of runtime context (operation queues, stats, subscribers, its URI entry in the host map). The total per service is about 300 bytes, so while tiny, for a fully indexed, highly available, scale out, asynchronous component, it can still potentially limit the total number of services instances (with each representing a configuration object or document in a control plane).

In order to make DCP be disk-bound only, not memory bound, a on demand pause/resume exists, which periodically calculates service host memory use, and persists runtime service context on a dedicated "blob" index. The service runtime context is removed completely.

# Pause

ServiceHost runs periodic maintenance and uses the setServiceRelativeMemoryLimit() API to determine the upper bound on its own memory use. The default bound is 50% of total JVM memory.
During maintenance the host does the following
 * Computes the total size, in MB of memory utilized by caching, operation queues, and per service runtime context. Some defaults are used to estimate memory cost, based on numbers derived from memory analysis tools
 * Retrieves the LOW_WATERMARK memory limit assigned to self (set by default or the user through setServiceRelativeMemoryLimitMB
 * If the total use is below the low watermark, no action is taken
 * If we are above the watermark, a fixed number of service instances are selected for pause

To pause a service, the following criteria have to be met:

 * The service has not seen any updates in the past few maintenance intervals, thus deemed inactive
 * The service is in AVAILABLE stage
 * The service has the PERSISTENCE option set

An eligible service will then be:
 * removed from the active dispatching map
 * set to ProcessingStage.PAUSED
 * Its service object instance will be serialized to bytes, using Utils.toBytes()
 * a POST will be sent to the /core/service-context-index ephemeral index store (so it will persisted to disk, and associated with a self link)

# Resume

Service pause is logically transparent to an external client (service or user). The only noticeable side effects are increased latency to complete the first operation received after a pause, and some additional per instance stats that document the number of pause/resume actions.

A service is resumed, transparently, when an operation is received by the service host. The host will

 * Query the service context blob index, using the self link as the primary key
 * If a blob, representing the serialized service exists, the service is deserialized into a java runtime object
 * The service is re-attached to the active dispatch map and set to ProcessingStage.AVAILABLE
 * The inbound operation that caused the resume is processed

# Validation
A long running test in our CI proves that a DCP service host process, limited to 64MB (or some other small number) can support millions of service instances, created at the rate of 1M per hour, and still run with over 80% of heap available. In addition, it randomly picks "paused" services and issues requests, to verify resume works