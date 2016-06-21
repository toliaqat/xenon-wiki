# Overview

A node selector service picks a node, among many in a node group, given a key and a set of criteria. The selection criteria are both supplied as the initial service state and are baked in the service code. The default node selector uses consistent hashing to assign a key (a document self link for example) to a set of nodes.

# Load balancing
The node selector is used by Xenon hosts to forward requests to each other, effectively building a peer to peer load balancer. A client can talk to any node and the request will be forwarded as appropriate.

The description of the hashing algorithm can be found [here](./leaderElectionAndReplicationDesignPage#leader-owner-selection-view-progression-reconfiguration)

# REST API

## URI

```
/core/node-selectors/default
/core/node-selectors/default-3x
```

## Selector instances

Notice that two node selectors started by default, by the service host:

 1. The default node selector, which is used implicitly for all replicated services, will replicate updates to all nodes and choose one node as the owner for a given selflink
 1. The limited replication node selector (3x replication) will replicate to 3 selected nodes per self link (the 3 closest ones in the euclidean sense) and just like the default selector, select a single owner per self link.

The limited replication selector (or any custom selector) is associated with a service by using this call, in the service constructor:

```
    public LimitedReplicationExampleService() {
        super(ExampleServiceState.class);
        toggleOption(ServiceOption.PERSISTENCE, true);
        toggleOption(ServiceOption.REPLICATION, true);
        toggleOption(ServiceOption.INSTRUMENTATION, true);
        toggleOption(ServiceOption.OWNER_SELECTION, true);
        super.setPeerNodeSelector(ServiceUriPaths.DEFAULT_3X_NODE_SELECTOR);
    }

```

The selector path can be set in the child service or the factory.

### Helper services

Every node selector instance must support the following helper services

```
/core/node-selectors/default/forwarding
/core/node-selectors/default/replication
/core/node-selectors/default/synch

```

## Forwarding service
The forwarding service takes a set of URI arguments and forwards a request to the selected node. Since its available on any node, it acts as the REST api to the L7 load balancing and hashing logic available through the node selector.

Xenon essentially has a built in load balancer in every node: if you issue a request to a node selector /forwarding service, it will take it as is, and forward it to a peer or broadcast it.

In the command link examples below note the quotes surrounding the URI: **you need to wrap the URI with quotes to avoid escaping by the shell**

For example if you want to retrieve the state of a service instance, across all peers, issue a broadcast:

```
curl "http://192.168.1.27:8000/core/node-selectors/default/forwarding?path=/core/management/process-log&target=ALL"
```
Notice the **target=ALL** parameter

If you want Xenon to route your request (GET, PATCH, etc) to a node that the path/link hashes to, use target=KEY_HASH:
```
curl "http://192.168.1.27:8000/core/node-selectors/default/forwarding?path=/core/management/process-log&target=KEY_HASH"
```

To target a specific peer node, by id, use target=PEER&peer=id:
```
curl "http://192.168.1.27:8000/core/node-selectors/default/forwarding?path=/core/examples&target=PEER_ID&peer=55016fd0-7cbb-45ee-bc0c-32767c49ec6"
```

### GET

The default node selector service is started by the service host, with the following initial state:

```
{
  "nodeGroupLink": "/core/node-groups/default",
  "documentVersion": 0,
  "documentKind": "com:vmware:dcp:common:NodeSelectorState",
  "documentSelfLink": "/core/node-selectors/default",
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "940038d5-0669-474e-9f7e-645922823f7c"
}
```

The limited replication selector is started with initial state indicating replication factor of 3

```
{
  "nodeGroupLink": "/core/node-groups/default",
  "replicationFactor": 3,
  "documentVersion": 0,
  "documentKind": "com:vmware:dcp:common:NodeSelectorState",
  "documentSelfLink": "/core/node-selectors/sha1-hash-3x",
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "host-1"
}
```

## Synchronization
The node selector synchronization service is associated with an instance of a [node selector service](./nodeSelectorService) and is invoked to synchronize document versions and document epoch between replicated service instances.

The synchronization process is invoked when a node is determined to be the new owner of a service, either on startup or due to node group changes noticed during the steady state replication protocol.

Please refer to the [replication protocol page](./leaderElectionAndReplicationDesignPage) for the overall replication and consensus scheme

### Synchronization REST API

### URI
```
/core/node-selectors/default/synch
/core/node-selectors/<customname>/synch
```

### POST
The service is stateless and performs the following operation, using the POST action

### SynchronizePeersRequest

```
    public static class SynchronizePeersRequest {
        public static final String KIND = Utils.buildKind(SynchronizePeersRequest.class);
        public ServiceDocument state;
        public EnumSet<ServiceOption> options;
        public String factoryLink;
        public boolean wasOwner;
        public boolean isOwner;
        public URI ownerNodeReference;
        public String ownerNodeId;
        public String kind = KIND;
    }
```

### Synchronization triggers

State synchronization can be triggered on demand or automatically when a node selector service notices node group changes:

* Client issues PATCH to /core/management with a SynchronizeWithPeers request
* Service code calls serviceHost.scheduleNodeGroupChangeMaintenance()
* A replicated, owner selected service notices that the documentOwner or documentEpoch in the request body does not match its calculation for owner and epoch. It will trigger synchronization with peers and then fail the original request, which will be retried by the node that received the client request

Factory services issues synchronization requests in batches, for their locally enumerated children. A factory service kicks of synchronization when it receives a maintenance operation with MaintenanceReason.NODE_GROUP_CHANGE

### Synchronization steps

A synchronization request is associated with one service link. Synchronization requests for different services execute in parallel. The following steps are taken:

* Broadcast a GET to the document index service of all peers, with documentSelfLink = serviceLink
* Compute the best state given the local state and those supplied by the peers. The best state is the state with the highest document version in the highest document epoch
* if local node is selected as owner for self link, 
  * Increment version and epoch
  * issue POST (if service does not exist on peer) or PUT to all peers that do not have best state
* if local node is not owner, and currently selected owner has state, DONE
* if local node is not owner and currently selected owner does NOT have state (new node joined) assume responsibility to act as owner, if it was previously the owner (before new node join) and perform state update across peers

# Node network partitioning

The synchronization logic is triggered whenever a node joins a group (regardless if its new or has been running for a while), or a node notices that all its peers became unavailable, then available at some later time.

The Xenon synchronization logic can automatically resolve the following cases, after the network partition is restored:

 * Regardless of node, any service with higher version than its peers, or only present on one node, will eithe rbe recreated on the remaining nodes, or its state will be synchronized across all instances
 * If a service is deleted on some nodes, but is (re)created or updated on some other nodes, with a resulting higher version and timestamp, the service will be recreated
 * If a service is deleted on some nodes, and the version after delete is the highest among all nodes, any instances of the service running in other nodes will be removed

In general, after two sets of nodes rejoin, their states should converge. If two service instances have been modified within the time epsilon, and have the same version, the service will be marked as in conflict (using a stat on the service instance, per node)



