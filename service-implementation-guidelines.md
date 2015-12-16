# Overview

The xenon service model enables lightweight services that can umber in the millions, and handle very large message loads. It is important however to follow certain guidelines to avoid excessive CPU usage or wasting memory. In addition, serious correctness errors will occur if the developer side steps the service lifecycle and state management model.

## State Management

### Class instance fields
Avoid instanced fields in the service class implementation. A service class should be 100% code (handlers).
Persisted services, extending **StatefulService** in particular will not function correctly if the code relies on state outside the state PODO. Any instance variables will be lost if the service is paused to disk, or if the host restarts. In addition, if the service is replicated, any class instance fields will **not** be replicated across nodes. 

### Nested Collection fields

Avoid large collections inside a state PODO. Xenon creates a new version with the entire state, every time an update occurs, so updating large collections generates a very large index load. Split collections into services, managed by factories, when possible

### Opt-in Indexing per field

Indexing cost goes up linearly with the number of services and fields per service document. If you only need to store some fields, but do not require indexing (you will not be querying over them), mark the field with IndexingOption.STORE_ONLY. Also, if the field is not used for determining equality, mark it IndexingOption.IGNORE_FROM_SIGNATURE

## CPU

### Request / Response Payloads

Avoid setting a large response body in a service handlers. If a service handles PATCH for example, and you dont need to return the entire state to the client, set the operation body to null, or to a minimal request PODO, then complete the operation. This can increase the per service I/O throughput by 2x, since the runtime avoids the cloning and serialization cost, in addition to the network overhead.

### Periodic maintenance

A service with ServiceOption.PERIODIC_MAINTENANCE will incur a runtime cost, every time maintenance runs. Be careful with the maintenance interval and use the Service.setMaintenanceIntervalMicros method to set the interval for your service, to the minimum required. Otherwise each service instance will be called on the default host interval which is 1 second. With millions of services, this can cause massive CPU load.

Periodic maintenance currently prevents a service from being paused to disk, so it has serious memory implications

### Custom thread pools

Avoid creating and managing per service instance thread pools. Xenon services should be 100% asynchronous and should never require their own threads. If you have no choice, create a stateless service with its own, fixed size thread pool, and use messaging to have it do work that is blocking. Use the ServiceHost.allocateExecutor() method to allocate a custom thread pool.


