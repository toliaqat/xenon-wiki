# Overview

Xenon provides an operationally simple model to federate several nodes (Xenon service hosts) to form a node group. A node group provides high availability, scale out and the ability to dynamically expand / contract.
This tutorial will walk through starting a couple of xenon hosts, each in their process, have them join, and then demonstrate replication by adding a service instance on one node, then finding it on all nodes.

## Prerequisites

* [Example Tutorial](./Example-Service-Tutorial)
* [Xenon Clustering deck](https://github.com/vmware/xenon/blob/master/contrib/docs/XenonClustering.pptx)

# Starting a host

In a terminal window, preferably within a xenon enlistment, type the following to build the latest xenon jars:

```
cd xenon
git pull
mvn package -DskipTests=true
```
The above should have produced a host jar that we can now start in the terminal. This host will have just core services plus the example factory service (see example tutorial). Now lets start one host, on port 8000. If that port is not available on your system, use a different one.
```
java -jar xenon-host/target/xenon-host-0.5.1-SNAPSHOT-jar-with-dependencies.jar --port=8000 --peerNodes=http://127.0.0.1:8000,http://127.0.0.1:8001
```
At a different terminal, start a second host, on a different port, making sure you supply the proper port in the peerNodes argument:
```
java -jar xenon-host/target/xenon-host-0.5.1-SNAPSHOT-jar-with-dependencies.jar --port=8001 --peerNodes=http://127.0.0.1:8000,http://127.0.0.1:8001
```
Notice we started the second host with --port=8001

## Node join at startup

In xenon you can either dynamically join a node group by sending a POST to the local (un-joined) node group service at /core/node-groups/default, or, you can have the xenon host join when it starts, by supplying the --peerNodes argument, with a list of URIs. To make startup scripts simple, you can supply the **same set of URIs, including the IP:PORT of the local host**. Xenon will ignore its self address but concurrently join through the other peer addresses.

For more details on node group maintenance, see the [node group service](./NodeGroupService)

When a node is starting you will see log output:
```
[0][I][1453320789994][DecentralizedControlPlaneHost:8000][startImpl][ServiceHost/2c6a499e listening on 127.0.0.1:8000]
[1][I][1453320789997][DecentralizedControlPlaneHost:8000][normalizePeerNodeList][Skipping peer http://127.0.0.1:8000, its us]
[2][I][1453320791195][DecentralizedControlPlaneHost:8000][lambda$4][Joined peer http://127.0.0.1:8001/core/node-groups/default]
[3][I][1453320791301][8000/core/node-groups/default][lambda$1][Synch quorum: 2. Sending POST to insert self (http://127.0.0.1:8000/core/node-groups/default) to peer http://127.0.0.1:8001/core/node-groups/default]
[4][I][1453320791302][8000/core/node-groups/default][handlePatch][Adding new peer 866041aa-e8a3-4cbf-ba49-8cf7c567f37d (http://127.0.0.1:8001/core/node-groups/default), status SYNCHRONIZING]
[5][I][1453320791302][8000/core/node-groups/default][handlePatch][State updated, merge with 866041aa-e8a3-4cbf-ba49-8cf7c567f37d, self 79418bdd-c8e4-4ce1-b98b-d073c9ab06c3, 1453320791301005]
[6][I][1453320797027][8000/core/node-groups/default][handlePatch][State updated, merge with 79418bdd-c8e4-4ce1-b98b-d073c9ab06c3, self 79418bdd-c8e4-4ce1-b98b-d073c9ab06c3, 1453320797027026]
```
## Node Group Service

The state of the node group, including node availability is available through the REST API of the node group service. Using curl or your browser, example the default node group state, on either node:

```
$ curl http://localhost:8000/core/node-groups/default
{
  "config": {
    "nodeRemovalDelayMicros": 3600000000,
    "stableGroupMaintenanceIntervalCount": 5
  },
  "nodes": {
    "e289f2ff-2fa1-42fb-9bb7-726120e16ec9": {
      "groupReference": "http://127.0.0.1:8000/core/node-groups/default",
      "status": "AVAILABLE",
      "options": [
        "PEER"
      ],
      "id": "e289f2ff-2fa1-42fb-9bb7-726120e16ec9",
      "membershipQuorum": 1,
      "synchQuorum": 2,
      "documentVersion": 2,
      "documentKind": "com:vmware:xenon:services:common:NodeState",
      "documentSelfLink": "/core/node-groups/default/e289f2ff-2fa1-42fb-9bb7-726120e16ec9",
      "documentUpdateTimeMicros": 1453337346482000,
      "documentExpirationTimeMicros": 0
    },
    "93ded02d-e8a8-4d6b-b4e6-199a4ec56ac8": {
      "groupReference": "http://127.0.0.1:8001/core/node-groups/default",
      "status": "AVAILABLE",
      "options": [
        "PEER"
      ],
      "id": "93ded02d-e8a8-4d6b-b4e6-199a4ec56ac8",
      "membershipQuorum": 1,
      "synchQuorum": 2,
      "documentVersion": 2,
      "documentKind": "com:vmware:xenon:services:common:NodeState",
      "documentSelfLink": "/core/node-groups/default/93ded02d-e8a8-4d6b-b4e6-199a4ec56ac8",
      "documentUpdateTimeMicros": 1453338005121037,
      "documentExpirationTimeMicros": 0
    }
  },
  "membershipUpdateTimeMicros": 1453337353308007,
  "documentVersion": 959,
  "documentKind": "com:vmware:xenon:services:common:NodeGroupService:NodeGroupState",
  "documentSelfLink": "/core/node-groups/default",
  "documentUpdateTimeMicros": 1453338005122007,
  "documentUpdateAction": "PATCH",
  "documentExpirationTimeMicros": 0,
  "documentOwner": "e289f2ff-2fa1-42fb-9bb7-726120e16ec9",
}

```
Notice that both nodes are listed, with status AVAILABLE:
```
"status": "AVAILABLE",
      "options": [
        "PEER"
      ],
```
The node group service uses random probing and state merges to maintain the group state without a large number of messages.

# Replication

We are going to create an example service instance, by issuing a POST to the /core/examples service factory, on one of the nodes. We first issue a GET to the factory to see if they have child services:

```
$ curl http://localhost:8000/core/examples
```
Node at port 8000 returns:

```
{
  "documentLinks": [],
  "documentCount": 0,
  "queryTimeMicros": 1,
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "e289f2ff-2fa1-42fb-9bb7-726120e16ec9"
}
```
We now check with node at port 8001:
```
$ curl http://localhost:8001/core/examples
{
  "documentLinks": [],
  "documentCount": 0,
  "queryTimeMicros": 1,
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "93ded02d-e8a8-4d6b-b4e6-199a4ec56ac8"
}
```

**Note**: A factory service computes its response by doing an index query. It then fills in the documentOwner with the id of its host. This is not related to ServiceOption.OWNER_SELECTION, which applies only to stateful services

Both nodes have no example services. Since the example service is marked replicated, a POST on the factory on one node will create a replica on all peer nodes.

```
$ curl -X POST -H "Content-type: application/json" -d '{"documentSelfLink":"one","name":"example one"}' http://localhost:8000/core/examples
```
Now we issue GET to the factory in each node and notice that we have the service present, in both:

```
$ curl http://localhost:8001/core/examples
{
  "documentLinks": [
    "/core/examples/one"
  ],
  "documentCount": 1,
  "queryTimeMicros": 5996,
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "93ded02d-e8a8-4d6b-b4e6-199a4ec56ac8"
}
```
```
$curl http://localhost:8000/core/examples
{
  "documentLinks": [
    "/core/examples/one"
  ],
  "documentCount": 1,
  "queryTimeMicros": 5997,
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "e289f2ff-2fa1-42fb-9bb7-726120e16ec9"
}
```
## Owner selection

Xenon will pick a "owner" node to route updates and GET requests for service instances marked with ServiceOption.OWNER_SELECTION. We can verify that the same owner id is returned for the example service we just created (the GET is actually automatically routed to the owner node, regardless of what entry node the client directs the request

```
$ curl http://localhost:8000/core/examples/one
{
  "keyValues": {},
  "name": "example one",
  "documentVersion": 0,
  "documentEpoch": 0,
  "documentKind": "com:vmware:xenon:services:common:ExampleService:ExampleServiceState",
  "documentSelfLink": "/core/examples/one",
  "documentUpdateTimeMicros": 1453337721979002,
  "documentUpdateAction": "POST",
  "documentExpirationTimeMicros": 0,
  "documentOwner": "e289f2ff-2fa1-42fb-9bb7-726120e16ec9",
  "documentTransactionId": ""
}

$ curl http://localhost:8001/core/examples/one
{
  "keyValues": {},
  "name": "example one",
  "documentVersion": 0,
  "documentEpoch": 0,
  "documentKind": "com:vmware:xenon:services:common:ExampleService:ExampleServiceState",
  "documentSelfLink": "/core/examples/one",
  "documentUpdateTimeMicros": 1453337721979002,
  "documentUpdateAction": "POST",
  "documentExpirationTimeMicros": 0,
  "documentOwner": "e289f2ff-2fa1-42fb-9bb7-726120e16ec9",
  "documentTransactionId": ""
}
```

Notice that the documentOwner points to node at port 8000. We did a GET earlier on /core/node-groups/default which provides the NodeState and information for each peer node. Here is the relevant portion for the node at port 8000:
```
...
"e289f2ff-2fa1-42fb-9bb7-726120e16ec9": {
      "groupReference": "http://127.0.0.1:8000/core/node-groups/default",
      "status": "AVAILABLE",
      "options": [
        "PEER"
      ],
      "id": "e289f2ff-2fa1-42fb-9bb7-726120e16ec9",
...
...
```

### Friendly node IDs

Instead of hard to remember class 4 UUIDs, you can specify a hostId when you start a Xenon host:
```
java -jar xenon-host/target/xenon-host-0.5.1-SNAPSHOT-jar-with-dependencies.jar --port=8001 --peerNodes=http://127.0.0.1:8000,http://127.0.0.1:8001 --id=hostAtPort8001
```
A GET to the node group now provides the following (we restarted both nodes after specifying the new --id parameter)
```
$ curl http://localhost:8000/core/node-groups/default
{
  "config": {
    "nodeRemovalDelayMicros": 3600000000,
    "stableGroupMaintenanceIntervalCount": 5
  },
  "nodes": {
    "hostAtPort8000": {
      "groupReference": "http://127.0.0.1:8000/core/node-groups/default",
      "status": "AVAILABLE",
      "options": [
        "PEER"
      ],
      "id": "hostAtPort8000",
      "membershipQuorum": 1,
      "synchQuorum": 2,
      "documentVersion": 2,
      "documentKind": "com:vmware:xenon:services:common:NodeState",
      "documentSelfLink": "/core/node-groups/default/hostAtPort8000",
      "documentUpdateTimeMicros": 1453338214662000,
      "documentExpirationTimeMicros": 0
    },
    "hostAtPort8001": {
      "groupReference": "http://127.0.0.1:8001/core/node-groups/default",
      "status": "AVAILABLE",
      "options": [
        "PEER"
      ],
      "id": "hostAtPort8001",
      "membershipQuorum": 1,
      "synchQuorum": 2,
      "documentVersion": 2,
      "documentKind": "com:vmware:xenon:services:common:NodeState",
      "documentSelfLink": "/core/node-groups/default/hostAtPort8001",
      "documentUpdateTimeMicros": 1453338279309033,
      "documentExpirationTimeMicros": 0
    }
  },
  "membershipUpdateTimeMicros": 1453338225178007,
  "documentVersion": 87,
  "documentKind": "com:vmware:xenon:services:common:NodeGroupService:NodeGroupState",
  "documentSelfLink": "/core/node-groups/default",
  "documentUpdateTimeMicros": 1453338279310007,
  "documentUpdateAction": "PATCH",
  "documentExpirationTimeMicros": 0,
  "documentOwner": "hostAtPort8000",
  "documentTransactionId": ""
}

```
