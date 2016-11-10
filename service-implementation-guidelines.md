# Overview

The xenon service model enables lightweight services that can number in the millions, and handle very large message loads. It is important however to follow certain guidelines to avoid excessive CPU usage or wasting memory. In addition, serious correctness errors will occur if the developer side steps the service lifecycle and state management model.

## Related content

 * [Example Service Tutorial](./Example-Service-Tutorial)
 * [Testing Guide](./Testing-Guide)

## Code Re-Use 

Xenon is not an object oriented model. Its a message passing, agent model that promotes re-use through service composition and stateless utility classes. Since its default implementation is java, it does rely on some very shallow inheritance but any heavy inheritance beyond the service base classes *should be avoided*.

Service should re-use code *not* through inheritance but through:
 * stateless helper services
 * singleton, stateless utility classes that take operations or completions (and never block)

Xenon services should look similar, and "flat". Each service should have handlers, finite state machine like methods, etc. See the tutorials for patterns

## Cross Service Interaction

There is a cost to patching / querying services so avoid tiny patches or queries across the entire index or node group. Think scale: If you have to send N messages to achieve an end goal, and each message is a index update or query, it will start having an impact in performance.

Always start with protocol design and interaction diagrams, and iterate as you develop services.

### Asynchronous only

Please review the asynchronous coordination tutorials and **avoid blocking/synchronous code** in xenon services at all costs:

 * [Deferred Result Tutorial](./DeferredResult-Tutorial)
 * [Coordinating async ops](./Coordinating-Async-Operations-(and-avoiding-callback-hell))

Avoid creating and managing per service instance thread pools. Xenon services should be 100% asynchronous and should never require their own threads. If you have no choice, create a stateless service with its own, fixed size thread pool, and use messaging to have it do work that is blocking. Use the ServiceHost.allocateExecutor() method to allocate a custom thread pool.

## State Management

### Class instance fields
Avoid instance fields in the service class implementation. A service class should be 100% code (handlers).
Persisted services, extending **StatefulService** in particular will not function correctly if the code relies on state outside the state PODO. Any instance variables will be lost if the service is paused to disk, or if the host restarts. In addition, if the service is replicated, any class instance fields will **not** be replicated across nodes. 

### Nested Collection fields

Avoid large collections inside a state PODO. Xenon creates a new version with the entire state, every time an update occurs, so updating large collections generates a very large index load. Split collections into services, managed by factories, when possible

### Opt-in Indexing per field

Indexing cost goes up linearly with the number of services and fields per service document. If you only need to store some fields, but do not require indexing (you will not be querying over them), mark the field with PropertyIndexingOption.STORE_ONLY. Also, if the field is not used for determining equality, mark it PropertyIndexingOption.EXCLUDE_FROM_SIGNATURE.

## CPU

### Request / Response Payloads

Avoid setting a large response body in a service handlers. If a service handles PATCH for example, and you don't need to return the entire state to the client, set the operation body to null, or to a minimal request PODO, then complete the operation. This can increase the per service I/O throughput by 2x, since the runtime avoids the cloning and serialization cost, in addition to the network overhead.

### Periodic maintenance

A service with ServiceOption.PERIODIC_MAINTENANCE will incur a runtime cost, every time maintenance runs. Be careful with the maintenance interval and use the Service.setMaintenanceIntervalMicros method to set the interval for your service, to the minimum required. Otherwise each service instance will be called on the default host interval which is 1 second. With millions of services, this can cause massive CPU load.

Periodic maintenance currently prevents a service from being paused to disk, so it has serious memory implications
