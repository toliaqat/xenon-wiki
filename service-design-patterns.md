# Overview

This guide  describes a few  common guidelines and  recommendations when
implementing  a Xenon  service. It  also  describes a  few common  service
patterns.

## Guidelines

When starting  on a new  service first  go through the  following steps,
which will help you generate the proper implementation:

1.  Draw two  interaction  diagrams. An  interaction  diagram shows  the
messages exchanged between services:

   a. A diagram  that describes how this service will  be used by others
   (its clients). Make sure to  include all self-modifying operations: A
   service should always issue a self PATCH or PUT if it needs to change
   its state, instead of doing it implicitly.

   b. A diagram that describes how this services orchestrates with other
   services, when it receives a client requests or does a periodic task

2. Describe  the service document,  the state PODO. List  the properties
and their type and note the  behavior of state side effecting operations
on each field.

3. Pick a pattern (more on patterns in a later section)

## Extensibility

Xenon promotes re-use  through services: If you have same  shared piece of
logic  or state  create a  service to  represent it,  and then  use that
service, through its REST API.

For small  stateless methods consider  a static class, invoked  with the
appropriate context  and a  operation, with  a nested  completion. These
"helpers" form a language-specific  library but since they are stateless
they compose well with asynchronous service code.

Avoid _inheritance_: Do  not sub class service  classes. The inheritance
model is  not composition and  will not  benefit an external  clients or
services in other  languages. It also has unknown semantics  in terms of
concurrency  and  operation life-cycle  and  creates  a "mini"  framework
within the inheritance chain.

## Data model design

If you have services that have relationships between each other, you can use links in their state documents to express them. Avoid bi directional links and rely on queries to find the relationships at runtime.

A good practice is to have links flow a single way: from leaf documents towards core documents. A core, common service document should never know (should never have links in its document) to all the consumers/leaf services that refer to it.

### Example for Data model references

A resource pool service is created at 
```
/resources/pools/east-coast
```

N Compute services, representing a physical machine are created at
```
/resources/computes/hostA
/resources/computes/hostB
/resources/computes/hostC
```

The document for compute has a link to the shared, root resource, the pool:
```
{
  "poolLink" : "/resources/pools/east-coast",
  "cpuCount" : 1,
}
```

## Patterns
 
*  _Is  the service  modeling  a  long-running  task?_ Its  document  is
something that describes  the task and it should  periodically update it
with progress. The service should include the TaskStae PODO in its state
and send self-PATCHes to move its  state machine forward. 

*  _Is the  service modeling  a  configuration object?_  It should  have
validation  in the  update  handlers and  set
OWNER_SELECTION, EAGER_CONSISTENCY  if strong replication  semantics are
required.

* _Is the service a stateless orchestrator?_ When receiving instructions
through a POST, and issuing  asynchronous requests to other services, it
should determine  if: (i) a task-like  service, which means it  needs to
reply right away with `201`; or (ii) if it is time-bounded, and is local
orchestration, which  means it  can do  the work in  the context  of the
client request and reply when the orchestration is complete.

* _Is this service managing a collection of documents/services?_ Then it should be split into two: 

  a. A factory service that creates new service instances for each document
  b. A "child" service representing each item.
   
