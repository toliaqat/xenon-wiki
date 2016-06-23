# Overview

Xenon addresses the need to build decentralized control and management plane components that are portable, lightweight (a few hundred bytes overhead per instance), fast, durable and observable. Each component is a tiny "service", listening on HTTP URI, representing some document + behavior. It can serve its own javascript for custom, persona oriented UI. The documents contain references to other services and properties that affect the behavior of the service. A common, replicated, indexed store allows for rich queries across service state, and the ability to <strong>dynamically</strong> compose service views.

The framework also address developer productivity and developer composition: developers can leverage common features that abstract complex third party technologies behind a consistent HTTP/REST API, and can leverage each others work without forcing dependencies (using HTTP mock services).

It should be noted that the Xenon framework abstracts the choice of replication, durability and cluster management allowing us to choose the right tool, at the right time and for the right job. We are starting with Lucene, because we have experience with it, but we could move to other document stores (in non replicated mode).

## Third party technology

Proven third party technology or research is used to satisfy some of the key requirements, avoiding duplication of effort: 

 * [Lucene](https://lucene.apache.org/core/) for local indexing (persistence) and query,
 * [Scalable gossip algorithm](http://www.cs.cornell.edu/~asdas/research/dsn02-swim.pdf) for node failure detection and membership tracking
 * [Kryo](https://github.com/EsotericSoftware/kryo) for efficient runtime data isolation and binary serialization
 * [Netty](https://github.com/netty/netty) for non blocking HTTP/HTTPS processing. 
 * [Swagger](http://swagger.io/) Optional, dynamic support for generating swagger documentation UI at run time

In each case, third party dependencies are abstracted behind a API and do not leak into the framework

## Requirements
* Isolation of execution context - a single pool of threads is used across all services in the same process and a service handler runs in an available thread, never sharing a call stack with another service
* Isolation of data - No singleton, writeable properties are allowed. Services can only exchange data through REST requests. The runtime efficiently (using Kryo) clones request and response bodies, before dispatching operations between services within the same process. 
* Portable across operating systems
* ***Naming and discovery*** - the document index functionality provides, as a first order concept, discovery of services, on any node, using a query. This is alot more than key/value based naming/discovery. Any combination of natural "keys" or queries can be used to find services
* **Managed state and concurrency** - service author does not do explicit locking or state management
* **Rate throttling and context sensitive back pressure** - The runtime calculates request throughput, per authorized user or custom tags associated with operations, and applies back pressure, allowing for graceful handling of bursts, uneven loads and preventing failure cascades (see the [programming model page](./Programming-Model#request-rate-throttling-back-pressure))
* Asynchronous, Composable, Isolated Components - Execution and state isolation managed by runtime, not developer. All interaction through asynchronous message passing with advanced features (rate throttling, intelligent queuing tied to state side effects, built-in analytics per component, operation)
* Standard framing and wire format - HTTP plus JSON serialization is used for all interaction between process / nodes.
* **Clustering support** - Groups of nodes can be associated per service, and different level of guarantees can be chosen per service. Node membership is maintained by a scaleable gossip layer (see SWIM paper reference) and replication is done by Xenon, on per update or batched mode. 
* **High Availability** - Symmetric or limited factor replication, selected per service using an eager or eventual consistency scheme. Read-after-write guarantees per service instance when owner selection is enabled.
* **Scale out with peer to peer forwarding** - Replicated, partitioned services automatically get tagged with an "owner" nodes and client requests are forwarded to the "owner" instance regardless of node entry point
* **Full document indexing** - The default store is a Lucene index, which tracks state versions per service. It also indexes each state version using the user name and referrer associated with the update, allowing for detailed auditing of state evolution. Due to symmetric replication a common indexed fabric is available for cross-service queries on any node
 * **Multi version state** - Durable services get each state indexed and persisted a unique version number allowing for efficient snapshots, rollback, observable state evolution 
* **Observable components** - The runtime provides HTTP endpoints, per service, to monitor health, support publication / subscriptions and provide example state for dynamic schema discovery
* **Built in analytics** - Using additional index service instances, events can be "pushed" and indexed at a very high rate, allowing for operation tracing, auditing, event logging and analysis. A analysis service provides realtime histogram of stats reduced and aggregated from incoming logs. In addition or instead of, a third party analytics service can be used.
* **Multi tenant/Sandboxed** - Each service host instance has its own view of local storage / index and its own HTTP/HTTPS root port. This allows for a high density multi tenant system, where multiple service host process run side by side on the same container, VM, OS, with no overlap. Cross host/tenant queries are STILL possible through the query API of per host index service.
* **Plugable AuthZ / AuthN** - Role base authentication services operating on users/user-groups, resources and verbs enable a simple but powerful authorization model, available across all Xenon nodes. Authentication is delegated to third party identity and single-sign-on providers
* **Disk limited, not memory limited** - Xenon implements on demand service instance pause/resume, so not only service state is kept on disk, but the service instance runtime context can be "paged out", on demand, allowing a Xenon host to have millions of service instances available, but be limited to just 64MB of heap! See [service pause / resume](./servicePauseResume) and [on demand load](./serviceOnDemandLoad)

## Service description
Each component, called a service, listens on a HTTP URI. The component has a set of capabilities, that indicate to the underlying runtime what its requirements are. The service author implements initialization and REST operation handlers.

The following links should provide context on Xenon design and on management functionality build as Xenon services

 * [programming model page](./Programming-Model) explains how to author a Xenon service, how to select various service options, and how REST actions map to Xenon service behavior.
 * [developer guide](./DeveloperGuide) describes how to start writing code in Xenon, workflow tools and the key programming interfaces in various languages
 * [service directory](./Service-Directory) contains a list of all core micro services and their documentation. This is the place to find higher level functionality such as provisioning, analytics, etc

### Declarative model
A developer simply declares the options a service needs, and the runtime implements them behind the scenes. Everything is 100% asynchronous using work stealing thread pools to balance work between service instances and processor cores.

In this [example](./Example-Service-Tutorial#example-service-code), a simple service implements a PATCH and PUT handler, updates state, and completes the operation. The current state, if any is supplied as part of the inbound request, loaded from the document store if not already cached.

The service instance is automatically replicated across Xenon nodes, and a consensus algorithm guarantees updates and GETs are routed + replicated appropriately.

The runtime automatically serves GET requests by serializing the latest state (the document) into JSON. It also serves requests directed to the utility services (/stats, /subscriptions, /template).

The runtime also provides per service [utility services](./REST-API#helper-services), simplifying both development and interaction with a service.

## Service discovery

The core principle in Xenon is that services have a URI path to identify them, and a document PODO to describe their state. Since every durable service instance is indexed, Xenon provides a very powerful discovery and naming service: Discovery is nothing more than a query, using the query task service, and the results are all the services that satisfy the query. So above and beyond a key value store, Xenon enables dynamic discovery using natural queries, not just primary keys

## Service state representation: The document

A service can be thought of as representing a live document, with validation code executing before any update occurs. Services can represent singleton configuration objects or dynamically composed collections of thousands of entities. 
Please review the [rest api page](./REST-API) for concrete examples on how state is represented (often referred to as the "document"), and additional helpers look like.

## Concurrency and State management
Xenon takes care of managing state, concurrency, metrics, and queueing of requests. Please see the [programming model](./Programming-Model) for details.

## Multi version state indexing
State changes never overwrite a previous change. Instead, they get indexed with a new version number, locally, and atomically computed. With each state update we index all the core service document fields plus
 * client ip address, 
 * version
 * owner node id for this service and version, 
 * authorization principal
 * update timestamp
 * SHA256 signature of content
 
Note that state version tracking can be bounded per service to last N versions or a storage quota. Xenon also tracks per service/document expiration.

# Decentralized design
The decentralized framework is an active-active system. Xenon uses consistent hashing and state replication to achieve both load balancing of "work" and high availability. Depending on service options different guarantees are given to a service instance.

See the [replication and leader election page for details](./Leader-Election-And-Replication-Design).
See also the [clustering slide deck](https://github.com/vmware/xenon/blob/master/contrib/docs/XenonClustering.pptx).

A brief summary of the key features:
 * High availability - If one service instance, in one node goes down, its peers will have seen the same update and indexed it locally. See ServiceOption.REPLICATION
 * Scale-out - Replicated service instances see a change made to a peer node, but the framework annotates their document with the owner node id, allowing all but the instance running on the owner node, to simply validate and complete the update. The instance on the owner can kick off any work, patching itself as it makes progress. See ServiceOption.OWNER_SELECTION
 * Strong consistency - A service with ServiceOption.OWNER_SELECTION can also enable ServiceOption.EAGER_CONSISTENCY to achieve strong consensus guarantees on every update. 
 * Peer to peer forwarding and load balancing - A client can target a service instance on any node, but the framework will forward the request to the node assigned ownership, without the client ever knowing. An asynchronous REST pattern is used implicitly so connections are not blocked waiting for peer responses. The client still observes the same API request/response pattern, as if talking to the instance on the owner node directly. See ServiceOption.OWNER_SELECTION

### Group membership
Services can elect to have state updates replicated across groups of peer nodes. A node is just the IP address and port of a service host, which makes replication work regardless of the underlying OS, container, VM or physical hardware.

The node group service is currently used to maintain the health and membership status between nodes, for a given node group. It uses a scalable random probing technique, based on the [SWIM paper](http://www.cs.cornell.edu/~asdas/research/dsn02-swim.pdf), to scale to large number of nodes. We rely on it so we can separate the problem of node health and failure detection, from that of state replication.

The control plane assumes symmetry: services that want to replicate state must have instances of themselves running across all nodes in a replication group.

# Service life cycle
A service can be created, updated, deleted, through HTTP methods. A developer can create a 10 line service implementation, package it and drop it in a runtime folder. The framework can support dynamic loading or require services linked in statically into one service host.
Assuming the code for a service is present, a "seed" collection service, often provided by the framework itself, is used to instantiate a service instance.

## Service on demand load/unload

Xenon implements a [pause/resume](./servicePauseResume) scheme so even the small runtime overhead of a service instance (less than 500 bytes) gets serialized and indexed on disk, in the Xenon blob index. This allows Xenon to be limited by disk size, per node, not available memory.

# Programming model

Please  see  the  [programming  model  page](./Programming-Model)  for
details on how a service is written,  how we map the REST API to service
handlers,  and what  the asynchronous  request pattern,  direct or  with
broadcast/forwarding looks like

# Performance

See [performance page (note its out of date)](./service-framework-performance). Please see the [lucene document index service page](./luceneDocumentIndexService) for indexing and query performance.

# References
* [Netty](http://netty.io/wiki/)
* [Kryo](https://github.com/EsotericSoftware/kryo#copyingcloning)
* [GSON](https://code.google.com/p/google-gson/)
* [Lucene](http://lucene.apache.org/)
* [Node gossip](http://www.cs.cornell.edu/~asdas/research/dsn02-swim.pdf)
