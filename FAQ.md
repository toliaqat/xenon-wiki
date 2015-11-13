# Xenon Frequently-Asked Questions

### 1. What languages are supported for writing services?

The service  runtime is currently  written in Java.  It can run  on JVMs
configured with  a maximum memory  pool (Xmx)  of 64MB and  _still_ host
thousands of service instances -- they are about 360-byte of working set
each, with state kept on index caches and disk, transparently.

The production code base is about 13K lines of code.

A basic  Go implementation of  the service host runtime  is implemented,
and multiple go  Xenon services exist, and a  Node.js implementation might
be desirable in the future.

### 2. Do services have descriptions or schema?

A  service document  is defined  by a  type. In  Java, that  would be  a
Plain-Old Data Object  (PODO) class, with fields, nested  PODOs etc. but
without any methods.  The runtime enables dynamic  discovery through the
use of `/template` (for any service instance, just append `/template` to
get  access to  the  parameters  each service  is  accepting, and  their
types). It provides an _instance_  of the service state, with additional
metadata (field types, field documentation, etc).

### 3. Where do I write business logic?

Business logic is encapsulated in  one or more cooperating services. For
example,  to write  a provisioning  service  for a  specific cloud,  you
re-use the same PODO (data model), write  a service, and have it talk to
cloud specific  provisioning APIs.  To other services,  or a  client, it
appears like any other provisioning service. 

### 4. What is the tenancy model?

Service  framework   instances  are   processes  that  can   run  inside
containers, VMs or native OSes.  The recommended setup is the following:
each tenant has its own process,  and an independent group of Xenon nodes,
which will give strong isolation in  terms of resources on disk, network
and  cpu. The  per-tenant node  group can  still be  part of  a provider
level node  group, monitored  by a  provider set  of service  hosts (its
recursive!). An alternative  is to use a single  management group, where
Xenon node  run service instances  across tenants. The  RBAC authorization
model  ensure isolation  between tenants  by giving  access to  specific
users and resources.
In additional the Xenon authorization model gives you runtime scoping on
all queries and requests to services using a query specification. If you
use fields common to documents to express tenancy, you can sandbox services,
per tenant this way. Please see the authorization page for more details

### 5. Is lucene exposed as a distributed, replicated store?

Lucene is  a library hidden behind a core Xenon service, to locally  to index
and store  updates. It is  not a  distributed store. All  replication is
done through Xenon mechanisms, entirely  independent of lucene. The Lucene
semantics are  hidden from  service implementation and  clients. Clients
always  send  updates  to  the  services, never  to  the  store  service
directly. Of  course, queries can be  sent to the document  store, which
will use Lucene underneath to satisfy complex, cross-document relational
queries. Both  persistence and  replication are implemented  as separate
services, transparently  working under  the covers  to make  Xenon service
documents durable and replicated across nodes -- but the Lucene document
store is not aware of any of this and is used as a local indexed store.

### 6. Does Xenon expose a key/value store? 

Among other things,  it does; but we  feel it is much more  than that: a
distributed, fully indexed, multi-versioned and composable collection of
service documents, that  use policies (control logic)  to support common
patterns across distributed applications. It  is a framework that allows
hosting  lightweight  services  across nodes,  and  selectively  provide
indexing,  durability, and  replication for  the documents  each service
represents -- allowing users to pick  the mixture of CAP properties they
desire. All interaction is done through  a service instance instead of a
store. This  is a  key difference with  other architectures  that expose
databases  and key-value  stores as  first class  components and  expect
orchestration to be hosted and run somewhere else.

### 7. Is replication achieved using PAXOS or a two phase commit protocol?
 
Service options determine replication and consistency behavior.
We use  a variant  of viewstamped replication  (VR) for
services marked with `EAGER_CONSISTENCY`. If a service is replicated and
marked with `OWNER_SELECTION`,  all requests are  forwarded to an owner  node, using
consistent  hashing  of  the service  URI
across the node  ids. Please the replication and leader election page for
details.

### 8. Are strong consistency models supported?

Yes, if a service author  enables the strong consistency capability. The
VR protocol, using consistent hashing to  elect a leader, is employed to
make sure the majority of the nodes  agree on each state update. See the
[protocol page](./leaderElectionAndReplicationDesignPage)

### 9. How do I view the data with out running my application?

The analogy comes from 3 tier applications where a database allows users 
to read/modify data with out the web server running.Xenon allows us to query 
the documents via “/core/query-tasks” with out the actual Service running.

### 10. How should I communicate to remote REST API?

There is no difference in communicating to a service, written in Xenon or 
otherwise. One can use the same programming constructs like Operation, 
sendRequest to achieve this.

### 11. How do I debug my application? 

Xenon offers several mechanisms to debug your application in development or 
production environments. Note that the multi version document store allows 
you to see per service state evolution. See the 
[Debugging and Troubleshooting page](./Debugging-and-Troubleshooting).
