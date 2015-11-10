# The Decentralized Control Plane

Welcome! This document (still a work-in-progress, really) will teach you
about **DCP**  (the Decentralized Control  Plane). A common  joke within
various groups  is that nobody can  summarize _what DCP is_  in just one
sentence -- well, here it is then:

> DCP is a developer-centric framework and associated programming model
> for building massively distributed, configurable applications.

What  does  this mean?  The  first  part  --framework-- means  that  DCP
bundles together  solutions to  a number of  common problems  that arise
when  building  distributed  applications.  For  instance,  it  provides
persistence, replication, availability,  consistency, parallelism, and a
number of other features -- all while allowing the flexibility of tuning
the guarantees of different components (hence, the _configurable_). That
is, DCP allows different services (running together) to place themselves
at different points  in the space described by the  CAP theorem (if this
does not make much sense, do not worry too much, just keep reading).

The second part --progamming model--  means that, to take full advantage
of  the framework,  developers will  need to  express their  computation
using specific abstractions that the framework provides (e.g., Services,
Documents). Put  differently, the  model is  _opinionated_ in  the sense
that, by having  the developer express computation in a  certain way, it
frees them from having to worry about distribution at scale.

Both the  framework and programming model  are not tied to  a particular
language.  Currently our  implementation  and discussion  is focused  on
Java; we have a less feature-rich implementation in Go; and expect other
implementations to follow (e.g., JavaScript).

The rest of  this document is organized as follows:  (i) a 5,000-ft view
of DCP, (ii) building an example service (the _echo_ service), (iii) the
anatomy of a request, (iv) a deeper look (e.g., verbs, options), (v) the
core  services  (v)implementation  details,  (vi)  possibly  interesting
results (e.g., performance, tweaking)

## A whirlwind tour of DCP

In this section we want to align some of the terminology and take a look
at some of DCP's main building blocks.

Users take  advantage of DCP  by building _micro-services_ 
(or  just  services).  In  the  DCP  model,  services  come  cheap,  and
constitute  the  basic  blocks  for  building  applications:  a  complex
distributed application comprises of a  number of services that interact
with  each other  via  HTTP. One  could possibly  think  of services  in
similar fashion  to threads  or processes in  conventional applications,
and the way they interact with each  other -- of course DCP services and
conventional  execution abstractions  have  many  more differences  than
commonalities.

Generally, a  service is a  _purely functional_ control logic  and comes
with a  URI other  services or  users can poke  to interact  with, using
conventional HTTP verbs. A service might  include state (we then call it
statefull), or not (statelss), it might persist state to durable storage
or not, it might.. -- well, you  get the idea. The important bit is that
for  all  these  services,  DCP  takes  care  of  replication,  routing,
availability, forwarding  and many other features,  transparently to the
service builder. The user (you, dear reader!) just needs to

Concretely, for each service there is a _service factory_ (factory) that
allows  users  to create  an  unlimited  number of  _service  instances_
(instances). The factory instantiates the service (we will see what this
means), and  instances are  the live entities  the user  interacts with.
Factories are  an example of  stateless services -- they  don't maintain
any state, but just respond to specific requests that arrive by creating
new, deleting or somehow manipulating instance services.

To make things more concrete, suppose we want to create a minimal `echo`
service  that  when poked,  it  returns  whatever was  inserted  latest.
How  would  this work?  Well,  details  aside  (what follows  is  almost
pseudocode), sending a POST request with a state object _s_ would update
the state's latest version to _s_. A subsequent GET would return _s_, as
well as some metadata (e.g. version number etc.):

```bash
PUT http://localhost:1234/echoes/aTinyEchoService '{"message": "a new message"}'
```
> 200 OK 

```bash
GET http://localhost:1234/echoes/aTinyEchoService 
```
> 200 OK {"message": "a new message"}

To expand  on the previous discussion,  The factory would listen  on the
URL `/echoes`, and sending a HTTP POST on this would create an instance,
say `/echoes/one`. Now sending an HTTP PUT on `echoes/one` would store a
message,  whereas sending  an  HTTP GET  `echoes/one`  would return  the
lastly stored message. Of course, interacting with `echo` instance `one`
is completely independent of interacting with, say, `two` or `three`. So
far  we haven't  talked about  distribution, naming,  persistence, state
type etc.  but we  will cover  them soon --  we just  want to  build our
intuition.

To summarize, services comprise of:

* some state (e.g., "message")
* a url (e.g., `echoes/one`)
* a control logic (e.g., on a GET just read contents of state)

To start them, one needs to interact with their factory.

But what do we mean by state?  DCP uses _documents_, or PODOs (Plain Old
Data Object), to store state. These are objects that have no methods and
get  serialized directly  to JSON  for  data interchange.  In the  above
example, the _typed_  version would be something  like `String message`,
while the JSON response from a GET would look like

```json
{
  "message": "This is a message"
}
```
(omiting DCP-related metadata, such as document version etc.).

DCP currently  takes advantage of Lucene,  a battle-tested, state-of-the
art indexing and storage engine; but it  is by no means attached to this
particular document indexing and storage engine. All per-node operations
are offloaded  to this  engine, that provides  highly-optimized document
actions.  For  instance,  state  can  be  written  to  durable  storage,
including _all  versions_ and queried using  associative, complex queries
on document metadata.

The URL (also referred to as _selfLink_) is how one refers to a service.
In  terms of  naming,  the convention  for factory  services  is to  get
assigned a plural  form. As clients interact with the  factory to create
instances, they can  pass a name to the factory,  which, if not assigned
already, will be the  name of the new instance. If  it is already taken,
an error code  will be sent back,  and if the client does  not specify a
name, a random 128-bit  hash is chosen. For example, in  the case of the
echo factory  described earlier,  the URL would  be something  along the
lines of:

```
http://22.231.113.64:1234/echoes
````

Then two possible instances (the second explicitely named upon creation)
```
http://22.231.113.64:1234/echoes/755c582b-eef4-4e96-8450-09e7227355af
http://22.231.113.64:1234/echoes/mynewecho
```

The control logic is expressed implementing  handlers to a subset of the
HTTP verbs  (GET, POST  and HEAD from  the HTTP/1.0  specification; PUT,
DELETE  from HTTP/1.1;  and  PATCH from  RFC  5789). In  the  case of  a
stateful service, the handler may consult  the state or assign new state
to it. Typically,  services try to maintain  the DCP-augmented semantics
of these verbs  (described in detail later). If a  particular handler is
not specified for a service, then the default handler is invoked.

**TODO: Explain HOST**
**TODO: Descibe verbs, since they are used later on**

A  note  on  programming  style:  developers on  DCP  write  code  in  a
completely asynchronous,  non-blocking style.  This stems from  the fact
that operations in  DCP are not aware of whether  they are communicating
with a  local or remote resource,  so they maintain the  same _feel_ for
both.  Typically,  when  invoking  an action,  instead  of  waiting  for
completion,  we  pass a  function  (frequently  completely anonymous  or
_lambda_) that will be invoked when  our action completes. This leads to
a series of nested callbacks that  called one after the other, while our
main code (or each function in the pipeline) returns immediately.

## Getting Started with DCP

Some of these can be part of the Appendix

### Prerequisites

### Installation details

### Contributing 

## Hello DCP!

Let's turn our previous example into  code (you can follow with your IDE
from the `samples` directory of DCP). Although very simple, this service
will let us expose and further  explore many of the core concepts behind
DCP. To have a minimum service, we would need to implement the following
(not necessarily in this order):

### Echo Service

This is  implementation of the  service instance -- including  state and
logic.

The  state is  encapsulated  in `EchoServiceState`,  extending the  base
document --  `ServiceDocument`. The  base document class  provides extra
fields (e.g., selfLink, version, owner) and core methods (e.g., cloning,
deltas). The Document for the echo  service consists of only a `message`
of type `String`.

```Java
public static class EchoServiceState extends ServiceDocument {
    public String message;
}
```

In terms  of logic, the  service itself extends  the `StatefullService`.
This comes with  a number of features, including default  handlers for a
number of operations  (e.g., PUT, GET) -- which, hapilly,  we don't need
to augment: PUT updates the state,  GET returns such a state. Therefore,
we  only implement  the  constructure, which  registers this  particular
state type with our service.

```Java
public class SampleSimpleEchoService extends StatefulService {
    public SampleSimpleEchoService() {
        super(EchoServiceState.class);
    }
}
```

Notably, at  this point  we can  toggle various  options options  in the
constructor (e.g.,  replication, ownership, persistence), which  we will
see soon.

We  also   need  to  create   the  factory   that  will  take   care  of
spawning  service  instances  of  this particular  type.  Stemming  from
`FactoryService`  (which again  provides useful  defaults), it  requires
overriding  the  method  that  creates   instances  (so  that  it  calls
our  service).  Also,  this  is  where we  pick  the  selfLink  for  the
factory  -- for  instance, the  factory  service could  be listening  on
`http://localhost:1234/samples/echoes/`. Our constructor  again needs to
register the state type.

```Java
public class SampleSimpleEchoFactoryService extends FactoryService {
    public static final String SELF_LINK = ServiceUriPaths.SAMPLES + "/echoes";
    public SampleSimpleEchoFactoryService() {
        super(SampleSimpleEchoService.EchoServiceState.class);
    }
    @Override
    public Service createServiceInstance() throws Throwable {
        return new SampleSimpleEchoService();
    }
}
```
**TODO: Explain Host**

Let's start playing with the service a little bit. After we launch the
host 

show what happens with json

### Ramping Up

Echo previous and dcpc

### Further Explorations

Other examples, views, queries -- where to look for

## The Anatomy of a Request

## A Deeper Technical Overview

## Core Services

### Queries

One  of the  most  interesting features  in  DCP is  the  ability to  do
queries. It features a very  powerful query calculus (e.g., hierarchical
queries, fuzzy searches, proximity and ranges) backed up by Lucene, with
queries running across multiple nodes and  the whole state of the system
(including higher-order  properties of documents). Queries  can target a
particular node group  (i.e., subset of the deployment)  and results can
be handled in a number of different ways:

* conventionally-asynchronous: as discussed previously, results are handled in the request callback function.
* fully-asynchronous: one request can sends the query specification, prompting launch (and returning a selflink), and a later request reads the results
* continuous: queries never end, but populate results in an event-driven fashion, even as new documents and state updates land into the system

Queries  can  request _pagination_  --i.e.,  dividing  the results  into
pages, and  requesting particular  pages-- or _full  expansion_ --where,
instead of serlf_links  full documents are returned.  They are submitted
to the  `/core/query-task` service, which is  responsible for analyzing,
broadcasting and  executing the query  (using other services,  too), and
composing the  results. `/core/local-query-tasks` are  node-local (i.e.,
not load-balanced  or replicated)  allowing concurrent  execution across
different nodes.

Queries are driven by a query specification:
```
// write a simple query and send it to the query-task service
```

Launching the query is simply done by posting to the query service. Note
the action  of POST: the semantics  of the operation express  that a new
task  instance  is  created `/core/query-task/new-task-id`  which  might
complete  immediately, take  a couple  of  days or  never finish  (i.e.,
continuous). Before we  dive into more complex queries,  the table below
outlines different options of _how_ the queries are executed.

## The User Interface

## Implementation Details

## Performance Results

* size / consumption of services (idle, heavy; statefull, stateless)
* requests
* self-hosting statistics

## Conclusion

* talk about the document abstraction as a ubiquitous enabler

## Metadata

Dump space for notes on the book itself.

TODO: Add pictures pictures pictures!
TODO: Explain CAP theorem (pictures) and give a tiny bit of technical overview
