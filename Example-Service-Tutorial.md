# Overview
This simple tutorial describes a very simple service, and its factory (the way to create new instances). Use it as a starting point in creating your own service. It also describes how to start the default Xenon host, or multiple hosts that synchronize with each other.

# Prerequisites

 * [programming model](./Programming-Model) for an overview of the Xenon API surface and model.

 * [debugging and troubleshooting](./Debugging-and-Troubleshooting#starting-a-host) for more details on starting Xenon, including starting multiple, replicated hosts.

 * [custom host and service creation page](./Hosting-Custom-Services-On-Xenon) if you want to create and host custom services.

 * [custom UI per service](./HostYourUi)

 * [developer guide](./Developer-Guide) for instructions on how to build, pre reqs, etc

# Starting a Xenon host

Start the example host, which listens on port 8000 (in the xenon-host directory):
```sh
$ java -jar xenon-host/target/xenon-host-*-with-dependencies.jar
```

## Xenon Container

Note that Xenon is packaged also a **[container](./Developer-Guide#docker-images)** 

Building and using Xenon containers will be documented soon. 

# The service factory
A client can create new instances of a service through another service, the service factory. A service factory mostly relies on framework for doing all the work and has the following properties:
 * it is stateless - A factory relies 100% on the document index or service host for keeping track of any child service instances. The lifecycle of a child is controlled by DELETE requests directly to the child.
 * GET requests are queries - The factory simply forwards a GET to its URI, to the document index, which returns all documents that their selfLink is prefixed by the factory URI

## Creating a factory

A developer can either derive from the FactoryService abstract class, or use a static helper to create a default instance:
 Below is a minimal, but still useful, factory:
```java
    /**
     * Create a default factory service that starts instances of this service on POST.
     * This method is optional, {@code FactoryService.create} can be used directly
     */
    public static FactoryService createFactory() {
        return FactoryService.createWithOptions(ExampleService.class, ExampleServiceState.class,
                EnumSet.of(ServiceOption.IDEMPOTENT_POST));
    }
```

Since factory code gives limited ability to change POST or GET processing, deriving from the factory class is not recommended. In particular, **avoid any state side effects across services** in replicated factory handlePost methods. Most of the logic that deals with service creation happens on POST completion, and on the owner node, so you will be violating key invariants.

### Starting a Factory service

The factory service is a singleton that should be started on host start:
```java
    @Override
    public ServiceHost start() throws Throwable {
        super.start();

        startDefaultCoreServicesSynchronously();

        setAuthorizationContext(this.getSystemAuthorizationContext());

        // Start the example service factory
        super.startFactory(ExampleService.class, ExampleService::createFactory);

       ...
       ...
```


### Per service Utility URIs
Each service comes with a set of Xenon provided [utility services](./REST-API#helper-services), listening under the service/* suffix. Use them to gather stats, subscribe, or interact with your service through a browser

## POST Handling
The factory service does not need to implement any handlers, if no additional logic or validation is required for processing POST requests. The _FactoryService_ super class takes care of POST handling, and also GET. The child service URI will be the composite of the factory URI (the prefix) and whatever selfLink path was supplied in the POST body. If no body was supplied, a random UUID will be used for the child.

### Durability
If the service instance state must survive host restarts, the child service instance must enable the ***ServiceOption.PERSISTENCE*** in its constructor. The runtime will then make sure every state change, including service creation is indexed on disk, and on restart, it will automatically re-create all child services, per factory and set the version numbers to the latest known version per child service instance

Assuming the service host is running locally (see the developer guide and debugging page), first do a GET on the example factory, to verify no example service instances are created:
```sh
$ curl http://localhost:8000/core/examples
```

Alternatively you can use the Xenon cli **xenonc**

```sh
$ export XENON=http://localhost:8000
$ xenonc get /core/examples
```

The factory responds with:

```json
{
  "documentLinks": [],
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "168c9af7-193d-4b6c-92c5-409df5b27752"
}
```

The response above means no documents / services are created yet.

### POST Example

Create a new instance by issuing a POST. If no body is supplied the child service will have a default initial state and a random selflink. 

```sh
$ curl -X POST -H "Content-type: application/json" -d '{"documentSelfLink":"niki","name":"n"}' http://localhost:8000/core/examples
```

Same operation, using xenonc
```sh
$ xenonc post /core/examples --documentSelfLink=niki --name=n
```

The example factory will respond with the initial state of the new service:

```json
{
  "keyValues": {},
  "name": "n",
  "documentVersion": 0,
  "documentEpoch": 0,
  "documentKind": "com:vmware:xenon:services:common:ExampleService:ExampleServiceState",
  "documentSelfLink": "/core/examples/niki",
  "documentSignature": "fdaccd2541efc842e42fb9693be00dbc731dea07",
  "documentUpdateTimeMicros": 1432745225186007,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "31716c5e-393b-4ca7-a15c-af02af0d6d5e"
}
```

Doing another GET on the factory URI shows the new child service

```sh
$ curl http://localhost:8000/core/examples
{
  "documentLinks": [
    "/core/examples/niki"
  ],
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "31716c5e-393b-4ca7-a15c-af02af0d6d5e"
}
```

## GET Handling
The super class handles GET requests to the factory URI and converts them to a document index query. The client can add the _expand_ URI parameter to request embedding the child service state as part of the result

# The child or singleton service

A service is called a singleton if its has a fixed URI and only one instance can exist per host. Otherwise, it just a child service, where many instances can co-exist under some shared URI prefix.

## Example Service Code

The example below is of a minimal service that represents a specific document (the ExampleServiceState PODO). The service implements a PUT and a PATCH handler for processing complete or partial updates to its state. Each time the state updates the framework indexes the new state, increments the version, content signature and timestamp. It also sends notifications, replicates if the service is replicated, etc

```java
public class ExampleService extends StatefulService {

    public static final String FACTORY_LINK = ServiceUriPaths.CORE + "/examples";

    /**
     * Create a default factory service that starts instances of this service on POST.
     * This method is optional, {@code FactoryService.create} can be used directly
     */
    public static FactoryService createFactory() {
        return FactoryService.createWithOptions(ExampleService.class, ExampleServiceState.class,
                EnumSet.of(ServiceOption.IDEMPOTENT_POST));
    }

    public static class ExampleServiceState extends ServiceDocument {
        public static final String FIELD_NAME_KEY_VALUES = "keyValues";

        public Map<String, String> keyValues = new HashMap<>();
        public Long counter;
        @UsageOption(option = PropertyUsageOption.AUTO_MERGE_IF_NOT_NULL)
        public String name;
    }

    public ExampleService() {
        super(ExampleServiceState.class);
        super.toggleOption(ServiceOption.PERSISTENCE, true);
        super.toggleOption(ServiceOption.REPLICATION, true);
        super.toggleOption(ServiceOption.INSTRUMENTATION, true);
        super.toggleOption(ServiceOption.OWNER_SELECTION, true);
    }

    @Override
    public void handleStart(Operation startPost) {
        if (startPost.hasBody()) {
            ExampleServiceState s = getBody(startPost);
            logFine("Initial name is %s", s.name);
        }
        startPost.complete();
    }

    @Override
    public void handlePut(Operation put) {
        ExampleServiceState newState = getBody(put);
        ExampleServiceState currentState = super.getState(put);

        // example of structural validation: check if the new state is acceptable
        if (currentState.name != null && newState.name == null) {
            put.fail(new IllegalArgumentException("name must be set"));
            return;
        }

        updateCounter(newState, currentState, false);

        // replace current state, with the body of the request, in one step
        super.setState(put, newState);
        put.complete();
    }

    @Override
    public void handlePatch(Operation patch) {
        ExampleServiceState updateState = updateState(patch);
        if (updateState == null) {
            return;
        }
        // updateState method already set the response body with the merged state
        patch.complete();
    }

    private ExampleServiceState updateState(Operation update) {
        // A Xenon service handler is state-less: Everything it needs is provided as part of the
        // of the operation. The body and latest state associated with the service are retrieved
        // below.
        ExampleServiceState body = getBody(update);
        ExampleServiceState currentState = getState(update);
        boolean hasStateChanged = Utils.mergeWithState(getDocumentTemplate().documentDescription,
                currentState, body);
        // update state

        updateCounter(body, currentState, hasStateChanged);

        if (body.keyValues != null && !body.keyValues.isEmpty()) {
            for (Entry<String, String> e : body.keyValues.entrySet()) {
                currentState.keyValues.put(e.getKey(), e.getValue());
            }
        }
        if (body.documentExpirationTimeMicros != 0) {
            currentState.documentExpirationTimeMicros = body.documentExpirationTimeMicros;
        }

        // response has latest, updated state
        update.setBody(currentState);
        return currentState;
    }

    private boolean updateCounter(ExampleServiceState body,
            ExampleServiceState currentState, boolean hasStateChanged) {
        if (body.counter != null) {
            if (currentState.counter == null) {
                currentState.counter = body.counter;
            }
            // deal with possible operation re-ordering by simply always
            // moving the counter up
            currentState.counter = Math.max(body.counter, currentState.counter);
            body.counter = currentState.counter;
            hasStateChanged = true;
        }
        return hasStateChanged;
    }

    /**
     * Provides a default instance of the service state and allows service author to specify
     * indexing and usage options, per service document property
     */
    @Override
    public ServiceDocument getDocumentTemplate() {
        ServiceDocument template = super.getDocumentTemplate();
        PropertyDescription pd = template.documentDescription.propertyDescriptions.get(
                ExampleServiceState.FIELD_NAME_KEY_VALUES);

        // instruct the index to deeply index the map
        pd.indexingOptions.add(PropertyIndexingOption.EXPAND);

        PropertyDescription pdName = template.documentDescription.propertyDescriptions.get(
                ExampleServiceState.FIELD_NAME_NAME);

        // instruct the index to enable SORT on this field.
        pdName.indexingOptions.add(PropertyIndexingOption.SORT);

        // instruct the index to only keep the most recent N versions
        template.documentDescription.versionRetentionLimit = ExampleServiceState.VERSION_RETENTION_LIMIT;
        return template;
    }
}
```

For an explanation of the options a service can declare, please see the [programming model page](./Programming-Model).

### PATCH handler
The PATCH handler above will use the body associated with the request (patch.getBody()) and the latest state of the service, associated with the patch (getState(patch)). Using fields in the patch body, it will update the current state and the complete the operation.

The service handlers are 100% asynchronous. This means that the handler method can exit, and the client **will not see the operation complete**. An operation is only completed when the service code calls
```java
operation.complete();
```
The service handler method can issue an asynchronous request to another service, return from the method, and only one the secondary request completes, complete its operation.
### CURL PATCH Example

Using curl once again, create a minimal JSON body with a name property and send it to the child service. Use the URI returned from the POST when creating the service, in the documentSelfLink field, or simply do a GET on the factory to get a list of example service instances

```sh
$ curl -X PATCH -H "Content-type: application/json" -d '{"name":"george"}' http://localhost:8000/core/examples/94639609-e989-4ffd-ade6-0a5f2841a421
```

Service responds with:

```json
{
  "keyValues": {},
  "name": "george",
  "documentVersion": 1,
  "documentKind": "com:vmware:xenon:services:common:ExampleService:ExampleServiceState",
  "documentSelfLink": "/core/examples/755c582b-eef4-4e96-8450-09e7227355af",
  "documentUpdateTimeMicros": 1411077167181000
}
```

The response has the new state, but you can also do a GET on the child service, to confirm the PATCH took effect. Notice the version is now 1.

# Stats

Xenon tracks per service instance, per operation statistics so you can determine latency, throughput, aid with debugging of your handlers. Please refer to the [REST API page](./REST-API#per-service-stats)