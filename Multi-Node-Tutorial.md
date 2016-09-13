# Overview

Xenon provides an operationally simple model to federate several nodes (Xenon service hosts) to form a node group. A node group provides high availability, scale out and the ability to dynamically expand / contract.
This tutorial will walk through starting a couple of xenon hosts, each in their process, have them join, and then demonstrate replication by adding a service instance on one node, then finding it on all nodes.

## Prerequisites

* [Example Tutorial](./Example-Service-Tutorial)
* [Xenon Clustering deck](https://github.com/vmware/xenon/blob/master/contrib/docs/XenonClustering.pptx)
* [Replication and Synchronization Algorithms](./Leader-Election-And-Replication-Design)
* [Multi node test setup, with Raspberry PIs!](./Multi-Node-Setup-with-Raspberry-PIs)
* [13. What will happen to accessing a service when node group is unstable?](FAQ#13-what-will-happen-to-accessing-a-service-when-node-group-is-unstable)
* [14. What will happen to the service when nodes are constantly up and down?](FAQ#14-what-will-happen-to-the-service-when-nodes-are-constantly-up-and-down)

# Starting a host

In a terminal window, preferably within a xenon enlistment, type the following to build the latest xenon jars:

```
cd xenon
git pull
mvn package -DskipTests=true
```
The above should have produced a host jar that we can now start in the terminal. This host will have just core services plus the example factory service (see example tutorial). Now lets start one host, on port 8000. If that port is not available on your system, use a different one.
```
java \
  -jar xenon-host/target/xenon-host-*-SNAPSHOT-jar-with-dependencies.jar \
  --port=8000 \
  --peerNodes=http://127.0.0.1:8000,http://127.0.0.1:8001
```
At a different terminal, start a second host, on a different port, making sure you supply the proper port in the peerNodes argument:
```
java \
  -jar xenon-host/target/xenon-host-*-SNAPSHOT-jar-with-dependencies.jar \
  --port=8001 \
  --peerNodes=http://127.0.0.1:8000,http://127.0.0.1:8001
```
Notice we started the second host with --port=8001

## Using HTTPS for Network Communication

Xenon provides the option to use HTTPS for Network Communication. This includes both, when a client communicates with a Xenon Host, and when one Xenon host communicates with another Xenon host (for Group Membership, Forwarding or Replication). This option is useful when running Xenon in an environment with untrusted clients that should be blocked from having direct access to a Xenon host.

To setup a Xenon host with HTTPS you will need the following: 
* An X.509 certificate chain file in PEM format.    
* A PKCS#8 private key file in PEM format.
* The password of the private key file (above), if it's password-protected.

Since each Xenon Host acts as both, a Listener for incoming requests and a Client for making requests to other Xenon-hosts, it is important that the X.509 Certificate(s) used for the rest of the Xenon-Hosts, should be registered with either the default trust-store on the machine, or you can create a custom trust-store and use that at Host start up time. 

For prototyping, you can use the certificate files checked-in in Xenon git-repo (below). These files are also used by Xenon tests:
* /xenon/xenon-common/src/test/resources/ssl/trustedcerts.jks
* /xenon/xenon-common/src/test/resources/ssl/server.crt
* /xenon/xenon-common/src/test/resources/ssl/server.pem

Once you have the files created, you can start a Xenon host with HTTPS communication by running the below commands:
```
java \
  -Djavax.net.ssl.trustStore=./xenon-common/src/test/resources/ssl/trustedcerts.jks \
  -Djavax.net.ssl.trustStorePassword=changeit \
  -jar xenon-host/target/xenon-host-*-with-dependencies.jar \
  --peerNodes=https://127.0.0.1:8000,https://127.0.0.1:8001 \
  --port=-1 --securePort=8000 \
  --keyFile=./xenon-common/src/test/resources/ssl/server.pem \
  --certificateFile=./xenon-common/src/test/resources/ssl/server.crt
```
At a different terminal, start a second host, on a different port, making sure you supply the proper port in the peerNodes argument:
```
java \
  -Djavax.net.ssl.trustStore=./xenon-common/src/test/resources/ssl/trustedcerts.jks \
  -Djavax.net.ssl.trustStorePassword=changeit \
  -jar xenon-host/target/xenon-host-*-with-dependencies.jar \
  --peerNodes=https://127.0.0.1:8000,https://127.0.0.1:8001 \
  --port=-1 --securePort=8001 \
  --keyFile=./xenon-common/src/test/resources/ssl/server.pem \
  --certificateFile=./xenon-common/src/test/resources/ssl/server.crt
```

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

### Friendly node IDs

Instead of hard to remember class 4 UUIDs, you can specify an ID when you start a Xenon host:
```
java -jar xenon-host/target/xenon-host-*-SNAPSHOT-jar-with-dependencies.jar --port=8001 --peerNodes=http://127.0.0.1:8000,http://127.0.0.1:8001 --id=hostAtPort8001
```

## Dynamic join and custom node groups (post-startup)

You can also create custom node groups after host startup, and have hosts dynamically join. This is useful if you want to have your Xenon services replicated (and be able to scale) independently from other Xenon services. 

For this section, we will assume you have two `Hosts` running locally; one on port 8000 (id: `hostAtPort8000`) and the other on 8001 (id: `hostAtPort8001`). This example will show you how to create a new `hello-node-group`, and have `hostAtPort8000` be a PEER, and have `hostAtPort8001` join as an OBSERVER. See the [NodeGroupService wiki](./NodeGroupService) if you're unfamiliar with these concepts. 

### Create a custom node group

Issue a `POST` against the `/core/node-groups` factory (just like any other Xenon service factory). You will need to create this service on both hosts.

```bash
$ cat hello-node-group.json 
{
  "config": {
    "nodeRemovalDelayMicros": 3600000000,
    "stableGroupMaintenanceIntervalCount": 5
  },
  "nodes": {},
  "membershipUpdateTimeMicros": 1456872059568025,
  "documentSelfLink": "hello-node-group"
}
$ curl -H "Content-Type: application/json" -X POST http://localhost:8000/core/node-groups --data @hello-node-group.json
$ curl -H "Content-Type: application/json" -X POST http://localhost:8001/core/node-groups --data @hello-node-group.json
```

> At this point, each host should have a `/core/node-groups/hello-node-group`, and each host's node-group should have only it's own host listed under `nodes`.

### Dynamically join another host's group as an OBSERVER

Next, you'll need to issue a `JoinPeerRequest`. We will have `hostAtPort8001` join `hostAtPort8000`'s group as an OBSERVER via a `POST` on the local `hello-node-service`.

```bash
$ cat observer-join.json 
{
    "kind": "com:vmware:xenon:services:common:NodeGroupService:JoinPeerRequest",
    "memberGroupReference": "http://127.0.0.1:8000/core/node-groups/hello-node-group",
    "membershipQuorum": 1,
    "localNodeOptions": [ "OBSERVER" ]
}
$ curl -H "Content-Type: application/json" -X POST http://localhost:8001/core/node-groups/hello-node-group --data @observer-join.json 
```

If this was successful, both `hostAtPort8000` and `hostAtPort8001` should return the same details for `nodes`. Also notice that `hostAtPort8001` joined as an OBSERVER.

```bash
$ curl http://localhost:8001/core/node-groups/hello-node-group
{
  "config": {
    "nodeRemovalDelayMicros": 3600000000,
    "stableGroupMaintenanceIntervalCount": 5
  },
  "nodes": {
    "hostAtPort8001": {
      "groupReference": "http://127.0.0.1:8001/core/node-groups/hello-node-group",
      "status": "AVAILABLE",
      "options": [
        "OBSERVER"
      ],
      "id": "hostAtPort8001",
      "membershipQuorum": 2,
      "documentVersion": 2,
      "documentKind": "com:vmware:xenon:services:common:NodeState",
      "documentSelfLink": "/core/node-groups/hello-node-group/hostAtPort8001",
      "documentUpdateTimeMicros": 1456883477064000,
      "documentExpirationTimeMicros": 0
    },
    "hostAtPort8000": {
      "groupReference": "http://127.0.0.1:8000/core/node-groups/hello-node-group",
      "status": "AVAILABLE",
      "options": [
        "PEER"
      ],
      "id": "hostAtPort8000",
      "membershipQuorum": 2,
      "documentVersion": 1,
      "documentKind": "com:vmware:xenon:services:common:NodeState",
      "documentSelfLink": "/core/node-groups/hello-node-group/hostAtPort8000",
      "documentUpdateTimeMicros": 1456883586893051,
      "documentExpirationTimeMicros": 0
    }
  },
  "membershipUpdateTimeMicros": 1456883477070005,
  "documentVersion": 771,
  "documentKind": "com:vmware:xenon:services:common:NodeGroupService:NodeGroupState",
  "documentSelfLink": "/core/node-groups/hello-node-group",
  "documentUpdateTimeMicros": 1456883586894013,
  "documentUpdateAction": "PATCH",
  "documentExpirationTimeMicros": 0,
  "documentOwner": "hostAtPort8001"
}
```

## Node Group Service

The state of the node group, including node availability is available through the REST API of the node group service. Using curl on the terminal, or your browser, issue a GET:

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

### Multiple node groups
A xenon host can be listed in multiple node groups. You can create new node groups through a POST to /core/node-groups, similar to any other factory service.

A service participates in a node group through a node selector service. Please refer to the slides / protocol page, linked at the top of this tutorial, for more details.

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
# Operational Model

## Node Group Expansion

* Add multiple nodes at a time, setting quorum to new majority, on all nodes (new and old) - This avoids wasteful rebalancing that can occur if a node is added one at a time. For example, if you want to expand a node group of 3 to, 5, add 2 nodes at once, but send an UpdateQuorumRequest to an existing node member first, with quorum set to 3. The new nodes should join with quorum already set to 3, using the JVM argument -Dxenon.NodeState.membershipQuorum
* Use blue-green update between old and new (bigger) clusters - Adding nodes to an existing cluster will invariably impact performance as rebalancing occurs. To minimize impact, the [live migration tasks](./Side-by-Side-Upgrade) can be used, first transferring state from a production node group, to a new, already established, stable node group.

