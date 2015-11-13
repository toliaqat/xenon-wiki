# Overview

Each  service usually  represents  a document  plus  some behavior.  The
document represents the  state, which can be  represented or transformed
by any  number of clients or  other services. The behavior  is triggered
from changes to state, either due to (i) external HTTP methods or (ii)
background periodic tasks.

Please also review the [service design patterns](./service-design-patterns).

## REST Actions (HTTP Verb mapping)

Review    the     [verb    semantics     section    in     the    design
page](./Programming-Model#verb-semantics-and-associated-actions)  on
how  Xenon maps  the well  known HTTP  verbs to  service state  update and
lifecycle actions.

## Utility services, per instance

Xenon starts a utility service  for every service instance.  The utility
service  listens on  a  set of  suffix paths  that  provide per  service
instance functionality:

 *  /stats -  Various stats  reported by  Xenon runtime  (see below),  per
 service and  custom stats reported  through the setStat  and adjustStat
 Service methods.

 * /ui - Custom, per service, UI rendered using either default JS + HTML
 files,  or files  found on  disk. UI  files should  be named  after the
 service name and be available on the local file system. For more, check
 the [UI info](./HostYourUi.markdown).

 * /config - Ability to change service options, and other configuration,
 at runtime.

 * /template  - Provides a default  instance of the service  state, plus
 per service document documentation,  options, comments. Its the dynamic
 PODO discovery mechanism

### Per service stats

Enable  the Capability.INSTRUMENTED  for  a service  and  then have  the
service  add stats  through out  its operation  handling and  lifecycle.
Xenon  provides all  stats through  the `/some-service/stats`  suffix. For
example, the node group service, that tracks node membership, uses stats
to indicate Serf RPC status.

```
$ curl http://192.168.1.121:8000/core/examples/97d9c342-b4ba-4b5f-b6e7-3f951e612b4d
```

The stats response:
```
{
  "entries": {
    "GEToperationQueueingLatencyMicros": {
      "name": "GEToperationQueueingLatencyMicros",
      "latestValue": 1.0,
      "accumulatedValue": 1.0,
      "version": 1,
      "lastUpdateMicrosUtc": 1426886948085003,
      "logHistogram": {
        "bins": [
          1,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0
        ]
      }
    },
    "stateCacheMissCount": {
      "name": "stateCacheMissCount",
      "latestValue": 1.0,
      "accumulatedValue": 0.0,
      "version": 1,
      "lastUpdateMicrosUtc": 1426886948083003
    },
    "GETrequestCount": {
      "name": "GETrequestCount",
      "latestValue": 1.0,
      "accumulatedValue": 0.0,
      "version": 1,
      "lastUpdateMicrosUtc": 1426886948083002
    },
    "GEToperationHandlerProcessingLatencyMicros": {
      "name": "GEToperationHandlerProcessingLatencyMicros",
      "latestValue": 2000.0,
      "accumulatedValue": 2000.0,
      "version": 1,
      "lastUpdateMicrosUtc": 1426886948085004,
      "logHistogram": {
        "bins": [
          0,
          0,
          0,
          1,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0
        ]
      }
    },
    "GEToperationDuration": {
      "name": "GEToperationDuration",
      "latestValue": 2002.0,
      "accumulatedValue": 2002.0,
      "version": 1,
      "lastUpdateMicrosUtc": 1426886948085005,
      "logHistogram": {
        "bins": [
          0,
          0,
          0,
          1,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0
        ]
      }
    }
  },
  "documentVersion": 0,
  "documentKind": "com:vmware:dcp:common:ServiceStats",
  "documentSelfLink": "/core/examples/97d9c342-b4ba-4b5f-b6e7-3f951e612b4d/stats",
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0
}
```

## State management

The framework  abstracts where  the document is  stored. If  the service
has  set   **ServiceOption.PERSISTENCE**,  the  state  will   be  loaded
asynchronously  and attached  to an  inbound request.  A service  author
should treat each handler invocation  as a completely stateless function
call:

 * The latest service state can be retrieved using the `getState(operation)` method.
 * The body of the request is retrieved using `request.getBody(type)`

_Because both  the state  and request  body are  passed as  arguments, a
service can  be shutdown,  migrated, change ownership,  updated, between
handler invocations with no extra care taken by the service author._

Its critical to  **avoid service class fields**.  Service classes should
have no fields, thus no cached values.

 The  underlying  document durability  layer  is  multi-versioned  which
 allows for conditional GETs (where a specific version can be specified)
 and conditional updates  (the update will be discarded  if the optional
 version specified does not match the latest version)

A  service  should  validate  all state  update  operations,  in  either
a  single  pass,  or  multi  phase  (more on  that  in  the  context  of
transactions).

A  service can  be entirely  stateless  but still  represent a  _dynamic
document_:  Operations on  them  are either  using information  gathered
asynchronously from other services or  are just compute bound, using the
operation body  alone to return  a response. The stateless  services can
compute a document, in the logical context of a GET.

## PODO service document common fields

The ServiceDocument  class is  extended by all  service state  plain old
data objects (PODOs) :

```  
    public class ServiceDocument {
            public String documentSignature;
            public long documentUpdateTimeMicros;
            public long documentExpirationTimeMicros;
            public long documentVersion;
            public String documentOwner;
            public String documentKind;
            public String documentSelfLink;
            public String documentSourceLink;
    }
```

These common fields allow the  framework to automatically do things such
as version  management, replication, partitioning,  snapshots, rollback,
audit trails.

 * selfLink - the unique identifier for the service and the URI path it listens to
 * version - a long value incremented atomically on all successful updates
 * updateTimeMicros - last update timestamp
 * expirationTimeMicros - optional expiration time. Document and service will be deleted if expiration passes
 * contentSignature - optional SHA256 signature of serialized content 
 * kind - a structured identifier that names the PODO type
 * owner - the node id of the node assigned to work on this document. Set by Xenon if ServiceOption.OWNER_SELECTION is enabled
 * sourceLink - Optional link to a document acting as the source for this document. Used when creating a new service through a POST to a factory that supports cloning

A service state document should be about data only, no behavior. In Java
and any language implementing the Xenon  programming model, it should be a
PLAIN-OLD-DATA_TYPE (only data fields, no methods)

```
    // An example service state PODO. 
    // The framework takes care of JSON transforms
    public class SampleServiceState extends ServiceDocument {
            public String uniqueId;
            public long counter;
            public double normalizedConsumption;
            public List<URI> otherServiceReferences;
            public Task pendingTask;
    }
```

Note that  the framework will  automatically index every field  and also
specially track  references between  state instances: This  enable cross
service relational  queries for  both client scenarios,  and consistency
checks (for  example, find out  if any  other service uses  an instance,
before deleting it)

## Collections

Collections are logical. A "factory" service  can be used to _create new
services_, but it does not "own" their content or life cycle. When a GET
is done at the root "factory" URI,  the framework simply does a query to
find the service self links with the same prefix as the factory service.
The GET returns  a list of self links. Using  URI query parameters, such
as `?expand=documentLinks`, the content within each child service can be
expanded and included  in the response, returning a  large collection of
documents.

The key  aspect of  keeping collections  logical is  that it  provides a
_dynamic_ query  based view of the  documents. A set of  services can be
logically organized and visualized  using just queries. Collection level
operations  (PUT, PATCH)  are executed  in  a transactional  way by  the
system, but  again, with out  requiring a concrete  "collection" service
implementation.

### ServiceOption.OWNER_SELECTION

The Xenon service host can automatically assign a Xenon node as the owner of
a new  service instance.  Given N  service instances  for child  item A,
across N nodes, only one of those instances will be routed updates, GETs
and will  have its handlers  invoked. The other replicas  simply persist
the  updates  using  the  replication  protocol.  See  the  [replication
protocol page for details](./leaderElectionAndReplicationDesignPage).

## Composition and extensibility

A micro service  is tied to a document, and  through references to other
service instances.  If a developer wants  to create a new  micro service
that  provides additional  functionality  to an  existing service,  they
simply  create  a new  PODO,  add  the new  properties  and  then add  a
reference property,  a URI, to the  existing micro service that  has the
existing fields.  A client can  issue a GET  to the new,  advanced micro
service, and  add a $expand directive,  and they will get  back one PODO
that includes both the old and the new fields.

## Query capabilities

Rich queries over all indexed documents can be issued through the [Query
Task Service](./queryTaskServiceDocumentation).  It provides a  task based
model for  specifying multi tier  queries (across nodes) with  a boolean
algebra composing various clauses.

### Example of composition
For example, imagine service A with the following document:

```
    GET virtual-machines/<guid>
    
    {
        name : "web server",
        status : "active",
        durationMicros : "122349820348"
    }
```

Now a new service wants to add a "storage node" property. It defines the
following PODO:

```
    GET advanced-virtual-machines/<guid>
    
    {
        storageNode : "http://somehost.org",
        vmReference : "http://<host>:<port>/virtual-machines/<guid>"
    }
```

By adding the "vmReference" property, it composed the new property, with
the _existing_ PODO and service implementation.

_TODO: How do you do this programatically?_

### Relationship between documents

The document store indexes  _relationships_ between documents. Any field
that is  a URI  will be  indexed in  a special  way, allowing  for graph
traversals of the document graph.

### Example of flattening relationships

Assume you have three logical collections of documents (and service endpoints)

1. /physical-machines/*
1. /virtual-machine-guests/*
1. /containers/*

A specific virtual machine document can be accessed like so

```
    GET /virtual-machine-guests/{id}
```

and can return a PODO that is _structurally_ similar to that of a physical
machine and a container, but with some additional fields.

Now using a query, you can find  all containers that are in that virtual
machine:

```
    GET /containers?$filter hostId eq {id}
```

The above  query, assuming there  is a  field "hostId" in  the container
document,  should  now return  all  containers  hosted by  that  virtual
machine.

### Counter example of using URI hierarchy instead of queries

The URI  hierarchy should  be flat. Deep  nesting creates  essentially a
deep query across a single pivot, that  can not be easily changed in the
future. In the above example, a _poor design choice_ would have been:

```
    GET /physical-machines/{id}/virtual-machine-guests/{id}/containers
```

## API Versioning

Once a REST API is published it should always be backwards compatible. This means:

* No fields can be removed or renamed
* If new fields are added  that replace in functionality odd fields, the
service must  take care  of converting  client requests  dynamically and
populating both  the old  and new fields  (but it can  only rely  on the
field when it loads from durable store)
* The URI namespace should **NOT** include the version number. For example, this is discouraged: `/v1/someapi` or `/someapi/v1`


## PODO Field/Property naming conventions

A properly  designed data model,  especially when exposed through  an HTTP
endpoint is  self documenting  and consistent.  The service  state PODOs
should follow these naming conventions depending on PODO property type:

* Simple collections - Property should be named "items". Example: `List<Tasks>` items;
* Maps - Property should be named "entries". Example: `Map<String,Tasks>` entries;
* URI - A single URI property should be end with (or contain) `reference`. Example: URI replicationServiceReference;
* Links - A single link property (service uri path) should end with/contain `link`.
* Multiple URIs - A collections of URIs should end with/contain `references` unless it is the only property, which in that case should end with/contain items: Example: List<URI> subscriberReferences;
* Date - A string representation of a date should end with/contain `date` or `time`;

## Helper services

Each service listens on a URI  (or multiple URIs) but the framework will
automatically add additional helper services, listening on URI suffixes,
that can provide additional functionality, with REST semantics.

 * `service/subscriptions` - Provides a list  of subscribers, as a list of
 URIs, which will be forwarded state update operations. This is the root
 mechanism of an efficient pub/sub model

 * `service/stats` -- if  the ServiceOption.INSTRUMENTATION  is enabled,
 provides a list of in memory stat entries, that can be modified, added,
 deleted by either the service  itself or remote entities. The framework
 will automatically report capabilities  and its own runtime information
 and performance stats through this REST endpoint, per service instance.
 Stat entries can be pushed to  the log service if indexing, persistence
 and analytics are required

 * `service/template` -- All services  are schema-less, but do provide a
 canonical, default representation of their state. The /template is auto
 generated from  the service PODO,  and allows for dynamic  discovery of
 the document instance, its fields, documentation, field types, etc

 * `service/config` -- Capabilities and configuration options per service
 instance

 *  `service/ui` --  HTML5  + Javascript  rendered  service current  and
 template state, with forms for submitting service updates. Available on
 every service instance, making  interaction through the browser (poking
 the service) simpler

## Examples

### GET example on singleton service
```    
    {  
       "enumerationServiceReference":"http://172.20.0.221:62041/common/node-groups/default",
       "enumerationAgentReference":"jsonrpc://172.20.0.221:62042",
       "systemInfo":{  
          "properties":{  
    
          },
          "environmentVariables":{  
    
          },
          "availableProcessorCount":0,
          "freeMemoryByteCount":0,
          "totalMemoryByteCount":0,
          "maxMemoryByteCount":0,
          "ipAddresses":[  
             "172.20.0.221"
          ]
       },
       "status":"UNKNOWN",
       "serverId":"87084a1d-8d77-4295-aef4-f6bb957995d1",
       "isRefreshRequired":false,
       "documentSignature":"4322961ce952655c9ba7345e448e556a602e2c934c6f56ee72d1c07fb85907fe",
       "documentUpdateTimeMicros":1407472558278011,
       "documentVersion":0,
       "documentKind":"com/dcentralizedsystems/services/peernodestate",
       "documentSelfLink":"/common/node-groups/default/87084a1d-8d77-4295-aef4-f6bb957995d1"
    }
```
### Collection GET (no expand)
``` 
    {  
       "documentLinks":[  
          "/common/node-groups/default/87084a1d-8d77-4295-aef4-f6bb957995d1",
          "/common/node-groups/default/d8e9b6d2-2e96-4864-98f0-27a4f90d5e63",
          "/common/node-groups/default/ca44f437-5333-4f9c-b526-6c5476f89796",
          "/common/node-groups/default/2f6cd8d1-36b9-4c2e-817d-c1200d650e57",
          "/common/node-groups/default/local",
          "/common/node-groups/default/3bd8c5c5-769d-4d91-9b25-cd85e1bdb627",
          "/common/node-groups/default/6e4ff81a-54ba-4ee8-bee8-9e58935dda7a",
          "/common/node-groups/default/ae758afb-bdcf-4bbb-8e98-c26a5afcfe6f",
          "/common/node-groups/default/bdb7a75c-1482-4b54-90b2-a045afd8ef55",
          "/common/node-groups/default/20c1d448-d294-48f4-917e-d0e7c6c41c2c"
       ],
       "documentUpdateTimeMicros":0,
       "documentVersion":0
    }
```
