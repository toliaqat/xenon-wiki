# Programming Model

The following types are the key components of the Xenon framework. Xenon is implemented primarily in Java, but a minimal and useful implementation of the service programming model is also available in Go, running in the go-dcp-* process.

 * ServiceHost - Manages the life cycle of micro service instances and is the unit of a Xenon node. Multiple hosts can co-exist within a process although production management hosts should be one per process, and one process per container. A host uses a ServiceRequestListener to receive inbound operations from the network
 * Service - The interface for a service implementation
 * StatefulService - The Java base class that handles most of the service workflow, allowing derived classes to simply override a few handlers and implement their specific validation and/or orchestration. This is the super class recommended for all service implementations
 * FactoryService - The stateless java base class that handles POST requests to create new services. It also handles GET, which is translated to a documentSelfLink prefix query, and returns all available children services
 * Operation - The container for a request / response message pattern. A service acting as a client to other services creates an operation, sets the URI, action (HTTP verb), body plus other fields, then uses the service client to send the request. A service handler is invoked when inbound operations are received from the network or other co-located services.
 * ServiceClient - The asynchronous client interface
 * HttpServiceClient - The implementation of ServiceClient for talking to other HTTP Xenon services or HTTP endpoints. The caller constructs an Operation instance, using a builder pattern, and calls send() on the client
 
## Asynchronous programming

Please review [coordinating async operations](./Coordinating-Async-Operations-(and-avoiding-callback-hell))

## Design patterns

The [design patterns](./service-design-patterns) page offers some tips on how to think about Xenon services and how to model various documents, workflows. It might be worth reading before diving into details 

## Implementation Guidelines

The [guidelines](./service-implementation-guidelines) page contains important implementation and performance guidelines. Please read before embarking on scale testing.

## Xenon Operation processing

### IO Pipeline

![io-pipeline](./io-pipeline.jpg)

### Service Creation (POST to Factory Service)

![Xenon-POST](./Xenon-POST.jpg)

### Update handling (PUT,PATCH, DELETE to a StatefulService)

![Xenon-PUT](./Xenon-PUT.jpg)

### Service Lifecycle

![service-lifecycle](./service-lifecycle.jpg)


## Service Interface
Please refer to [Service.java]
(https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/common/Service.java)

The best way to see the declarative nature of xenon services, and how an author can select which aspects of C.A.P they want to enforce, please read the tutorials.

## Service Creation

A service instance is created by issuing a POST to a factory service. Factory services should be started during service host start and are considered singletons. A factory service will create a new "child" service instance and ask the service host to start it. 

### Factory Service Handlers

Factory services are stateless so authors should keep them as simple as possible and defer initial state validation to the child service. The child service can do initial state validation in handleStart(). If the child service models a task, it should do validation as part of its task state machine. The handleStart() method should simple, initial state validation only (see the design patterns page) 


## Service Handlers
The main customization point is the service handler. That is where specific validation or orchestration logic lives. A service must override handlers for state update operations or to kick off any work. It does not need to implement a GET handler, or handlers for its utility functions (subcriptions, stats, template).

All updates are through HTTP methods and execute in arbitrary thread context. A HTTP GET provides the document representation, with the default being JSON serialization of the service state. The default GET response is implemented by the base service class (`StatefulService` in Java).

The runtime invokes a method handler (handleGet, handlePost, etc) passing it a Operation instance. The handler must be 100% asynchronous: no blocking I/O is allowed since the runtime uses just a few threads per core to schedule handlers across all components.

When a handler is finished processing a inbound request, it calls operation.complete() or operation.fail(), from any thread context. Before doing so, if a response body is expected, it must call operation.setBody().

The framework makes the handler a completely stateless method: It passes the operation (which contains the request body) and associated with it, the authoritative, latest state. The state is cloned before its passed to the handler, and its cloned again after the operation is complete (in any thread context, not just when the handler method returns)

### Request processing stages
1. If the current state is not cached, it will asynchronously load the latest state (the one with the highest update time and version) and associate it with the inbound operation. The service author can access the state using *operation.getLinkedState()*
2. The body is automatically converted to an in memory instance, using *operation.getBody(type)*
3. When the service author calls *operation.complete()* the framework will
   1. Update the state version atomically
   1. Update the update time
   1. Make sure key fields like self link and kind are set in the linked state
   1. If the service is replicated, the consensus and replication protocol will be invoked 
   1. If the service is indexed or durable, it will send the state to the document store for indexing and durability
   1. If there are any subscribers, it will issue the request, with the same body, to all subscribers
 
### Request rate throttling (back pressure)

The runtime supports back pressure, in the form of operation throughput tracking, per authorized user. The developer can adjust request rate limits by using the ServiceHost.setRequestLimit() methods.

```
// userPath is a link to a service presenting a user. It needs to much the subject
// in the Claims associated with inbound requests
 this.host.setRequestRateLimit(userPath, 1.0);

```

The request limits are computed over the host maintenance window and are expressed in operations per second. The runtime keeps track of operations / second for a given authorized subject. The rate is computed and averaged over a maintenance window, and reset at the start of the next.

If the compute average exceeds the rate limit, the request is failed with the appropriate error code (STATUS_CODE_UNAVAILABLE) and a suggested retry time is included in the response header.

See the verification test for an [example](https://github.com/vmware/xenon/blob/master/xenon-common/src/test/java/com/vmware/xenon/common/TestServiceHost.java#L116)


### Request Body Routing

Xenon can route to handler methods based on the action and the request body, if a request router is configured and set. See the [request routing](./custom-request-routing) for more details.

### Concurrency control
A Xenon service author should never have to:
1) Use locks or synchronization primitives. Since the latest state is cloned, and associated with each request, no side effects are visible between handlers. Further, the default behavior is to only have a single update active, for a specific service instance. A queue is used to serialized updates. GET requests are processed in parallel with update
2) Keep in memory fields. The service class should have no mutable fields that affect behavior. Services can migrate between handler invocations, be restarted, etc so all state should be in the service document associated with the service.

### Self update
It is very important to keep a symmetric model when it comes to state updates: Just like an external client has to issue a PATCH/POST/PUT to update state and initiate some work flow, so does any internal code. The developer of a service should just create an operation, and post it to its service URI. This makes all transitions traceable, observable and deals with nasty concurrency problems since the runtime will take care of queueing and synchronization.

### State update hints
If a service notices that an update operation does not provide new content, it can communicate to the framework to *not* update the current version and index a new state version. To do so it must do the following:

```
    operation
        .setStatusCode(Operation.STATUS_CODE_NOT_MODIFIED)
        .setBody(null);
```

The following handler implements PATCH and updates its current state using the body in the request. However, if it does not notice a difference between current and the patch, it set the STATUS_CODE_NOT_MODIFIED in the operation

```
    
    public void handlePatch(Operation patch) {   
        NodeState currentState = getState(patch);
        NodeState body = patch.getBody(NodeState.class);
        
        if (body.status != null) {
            if (body.status == currentState.status) {
                isNew = false;
            }
            currentState.status = body.status;
        }
        if (!isNew) {
            // Optimization: prevent a new version from getting indexed.
            patch.setStatusCode(Operation.STATUS_CODE_NOT_MODIFIED);
            currentState = null;
        }
        patch.setBody(currentState).complete();
    }
```

## Periodic maintenance

Services can opt in to receive periodic messages, in their **handleMaintenance** handler, so they can execute grooming, health and other periodic tasks. See ServiceOption.PERIODIC_MAINTENANCE. This option combines with ServiceOption.OWNER_SELECTION so only service instance designated as owners execute the potentially resource consuming tasks, allowing load balancing out among peers.

If an implementation of the **handleMaintence** method needs to interact with the state of its service, it must do so as if it were a client to that service.

i.e. 

* The state must be retrieved by requesting a `GET`.
* The state must be mutated by submitting a `PUT` or `PATCH`.

## Verb semantics and associated actions

* POST (collection) - If done on a collection, creates a new service instance, with the POST body being the initial service state. If service is durable, state is loaded automatically from the store/index. Its strongly recommended to avoid "sub collections" within a service document. A collection item should be a service
* POST - If done on a stateless service, it can be used to compute work, or to add new resources on the service. 
* DELETE - The service will be stopped. A persisted service will receive a new "empty" state version, with just the common fields. This will mark the self link associated with the service as "deleted". No state versions are actually deleted, the store will just fail GET requests for self links marked "deleted". Client can still request specific versions. A grooming cycle on the store can remove versions outside version retention limit for the service
* PATCH - Used to update one or more fields in the document represented by the service. The framework will load existing state, supply it as part of the inbound operation, and service author simple validates and "merges" the body of the operation with the existing state. On operation completion a new state instance is indexed and stored. PATCH can be used for atomic increment / decrement operations on counters tracked within a document
* PUT - A new document / service state, in its entirety, is supplied to a running service. The service validates and replaces its existing state, atomically
* GET - Retrieves the state of a service (the document). If a GET is done on a collection service, a query is executed and a paginated result of all the singleton (child) service URIs is returned.A GET on a stateless service might lead to several asynchronous operations to other services, to collection their documents, which the stateless service composes on the fly, as its state.

The framework allows a service developer to refine the semantics on a per service basis by authoring implementation for logical operations and registering those operations through a [RequestRouter](./custom-request-routing).

# Client API
The framework provides a 100% asynchronous API, with a builder pattern for creating operations. The design is language agnostic, but the examples given below are in Java, since that is the reference implementation. 

See [coordinating async operations](./Coordinating-Async-Operations-(and-avoiding-callback-hell))

## Point to point operations
The example below shows how to send N requests, to N services, in parallel (no blocking). When each request completes, a completion is invoked. If it fails the completion is passed in the error.
```java
TestServiceState newState = buildBody();
// send in parallel a request per service
for (URI service : services) {
    newState.id = UUID.randomId().toString();
    // sendRequest() does not block. 
    sendRequest(Operation.createPut(uri)
        .setBody(newState)
        .setCompletion((operation, failure) -> {
                if (failure != null) {
                    logSevere(failure);
                    return;
                } 
                // proceed with next step, using response body
                doSomething(operation.getBody(TestServiceState.class));
        }));
}
```
To parallelize outbound requests to even the same service host, an asynchronous client connection manager allows for multiple concurrent outbound connections, queuing up requests transparently when the connection limit is reached.

## Subscriptions

Xenon supports a publication / subscription model per service instance. A HTTP client can POST to a service /subscriptions utility suffix, with a body that includes the subscribers callback URI
```

    public static class ServiceSubscriber extends ServiceDocument {
        public URI reference;
        public boolean replayState;
        public Long notificationLimit;
    }
```

As a payload to the HTTP POST:

```
{
  "reference":"http://localhost:8000/notification-target"
}
    
```

The service host class provides a method to register for subscriptions. To subscribe:

 * create a POST operation, with the URI set to the service you want to subscribe
 * create a lambda to receive a callback for each notification (Consumer<Operation> delegate)
 * call serviceHost.startSubscriptionService

```
Consumer<Operation> target = (notifyOp) -> {
    logInfo("got %s notification", notifyOp.getAction());
    notifyOp.complete();
};

Operation subPost = Operation
    .createPost(UriUtils.buildUri(getHost(), someServiceLink))
    .setReferer(getUri());
getHost().startSubscriptionService(subPost, target);

```

Xenon supports auto deletion of subscriptions in the following cases:
 * the ServiceSubscriber.notificationLimit is set. The publisher service will delete the subscription after N notifications are sent
 * the ServiceSubscriber.documentExpirationMicros is set. The publisher service will delete the subscription when expiration is set
 * K consecutive attempts to publish a notification to a subscriber have failed. The publisher will delete the subscription

If expiration or notification limit is not set, the client (subscriber) should stop the subscription when its no longer needed, otherwise it will consume resources.

### Reliable subscriptions
Subscriptions are NOT persisted at the publisher. If the publishing service restarts, the subscriptions will be gone and clients need to re subscribe. Subscription requests will be routed to the owner for the service instance but if that owner changes, the new owner will not carry over the subscriptions. To address this, a client side reliable subscription service monitors node group changes and resubscribes to the publisher service, if it notices its subscription is missing.


To create a reliable subscription, the following method is used:

```java

    /**
     * Start a {@code ReliableSubscriptionService} service and using it as the target, subscribe to the
     * service specified in the subscribe operation URI
     */
    public URI startReliableSubscriptionService(
            Operation subscribe,
            Consumer<Operation> notificationConsumer) 

```

You can also create an instance of reliable subscription service then call this method:

```
    /**
     * Start the specified subscription service (if not already started) and specify it as the
     * subscriber to the service specified in the subscribe operation URI
     */
    public URI startSubscriptionService(
            Operation subscribe,
            Service notificationTarget,
            ServiceSubscriber request) 
```



## Join Operations 
The framework supports operation join pattern:
* If there are discrete completions, associated with each joined operation, each one is called at the end when all N operations are complete (K success + M failures == N)
* If there is a single completion (on just one of the operations, or same across all the operations) it's called when all operations are complete.
Since the same completionHandler is called, with the same arguments (o,e) -> {} the discrete operations can still be recovered like this example:

```java
Operation op1 = Operation.createPatch(computeFactory);
Operation op2 = Operation.createPatch(adapter);
Operation op3 = Operation.createPatch(networkFactory);

CompletionHandler ch = (o,e) -> {
    if (e != null) {
         //NOTE: the other completion handler might throw exception that is not handle there
         //In such cases use OperationJoin.create(op1, op2, op3) with setting shared completion.
         return;
    }

    for(Operation op : o.getJoinedOperations()) {
         // collect bodies across them do something. Check if its failed
    }

    // alternativaly, look up a specific operation (pre cloning)
    o.getJoinedOperation(op3).getBody(NetworkState.class);
}

sendRequest(op1.joinWith(op2).joinWith(op3).setCompletion(ch));
```

* Another way to invoke operations in parallel is to use the similar pattern OperationJoin.create(op1,op2,op3). The advantage in this way is that there is one shared completion, which will reduce the complexity with error handling and not require additional completionHandlers for every operation. Example usage: 
```java
     OperationJoin operationJoin =OperationJoin.create(op1, op2, op3)
         .setCompletion((ops,exc) -> {
            if (exc != null) {
                //it is a map with Operation.getId() as key and the related exception if any.
                Map<Long, Throwable> exceptions = exc; 
                //loop via the exceptions and do something with them. 
                return;
            }
            
            for(Operation op : ops.values()) {
                // collect bodies across operations do something. 
                NetworkState ns = op.getBody(NetworkState.class);
            }
         });
      operationJoin.sendWith(host);
```

## Sequence Operations
The framework supports operation sequence pattern where a set of operations are executed after previous set of operations are completed successfully. 

* Example usage:
```java
  OperationSequence.create(op1, op2, op3)
                    .next(op4, op5, op6)
                    .setCompletion((ops, exs) -> { // shared completion handler
                          if (exs != null) {
                               return;
                          }

                         Operation opr1 = ops.get(op1.getId());
                         Operation opr4 = ops.get(op4.getId());
                          // ....
                    })
                  .sendWith(host);
```

* Advanced example usage:
```java
OperationSequence.create(op1, op2, op3) // initial joined operations to be executed in parallel
            .setCompletion((ops, exs) -> { //shared completion handler for the first level (optional)
               if (exs != null) {
                   // Map<Long,Throwable> exceptions = exc;
                   for(Throwable e:exc.values()){
                       //log exceptions or something else.
                    }

                    // NOTE: if there is at least one exception on the current level
                    // the next level will not be executed.
                    // In case, the next level should proceed the exception map
                    // should be cleared: exc.clear()
                    // This might lead to inconsistent data in the next level completions.

                    return;
                }

                // Map<Long,Operation> operations = ops;
                Operation opr1 = ops.get(op1.getId());
                SomeState body = opr1.getBody(SomeState.class);

                // Can set properties on the operations in the next levels
                NextState nextStateBody = new NextState();
                nextState.property = body.otherProperty;

                op4.setUri(body.selfLink);
                op4.setBody(nextStateBody);
            })
            // next level of parallel operation to be executed after the first level operations
            // are completed first.
            .next(op4, op5, op6)
            .setCompletion((ops, exs) -> { // shared completion handler for the second level (optional)
                   if (exs != null) {
                      return;
                   }

                   Operation opr4 = ops.get(op4.getId());

                   // have access to the first level completed operations
                   Operation opr1 = ops.get(op1.getId());
             })
             .next(op7, op8, op9)
             .setCompletion((ops, exs) -> {
                 // shared completion handler for the third level (optional)
                 // all previously completed operations are accessible.
                 Operation opr1 = ops.get(op1.getId());
                 Operation opr4 = ops.get(op4.getId());
                 Operation opr7 = ops.get(op7.getId());
                 // In many cases, the last shared completion could be the only one needed.
             })
             .sendWith(host);
 }
```

## Group Operations
In addition, Xenon provides a broadcast-with-quorum facility. Using a single operation, and a group of nodes, it will send the request to all of them, asynchronously and then the client can decide if they act on the first, fastest response, or when quorum is received (multiple responses that pass some matching criteria)

See the **ServiceHost.broadcastRequest** method, and also the REST API available for broadcast, on **/core/groups/<group>/forwarding** URI (per node group).

Broadcast operations should only be used on non replicated services.

## Peer to peer forwarding
The forwarding functionality allows a client to send a request to Xenon node, and have that node transparently forward it to the designated owner of a service, or to the node that is assigned (through consistent hashing) to a key supplied by the client. See the **ServiceHost.forwardRequest** method and also the REST API available for peer to peer forwarding, on **/core/groups/<group>/forwarding** URI (per node group)

## Coordinating Asynch Operations
Please reference: [Coordinating-Async-Operations-(and-avoiding-callback-hell)](Coordinating-Async-Operations-(and-avoiding-callback-hell))