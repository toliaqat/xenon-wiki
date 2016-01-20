# Overview
The node group service tracks a group of Xenon nodes. It does failure detection and membership update between all nodes using an implementation of the SWIM protocol. 

The role of this service is described in the [design page](./Design#active-update-replication) and [replication protocol page](./Leader-Election-And-Replication-Design)

Multiple groups can co exist and run independently of each other. A service is implicitly associated with a group, through a [node selector service](./NodeSelectorService)

A visual description is in this [deck](https://github.com/vmware/xenon/blob/master/contrib/docs/XenonClustering.pptx)

## Nodes

A Xenon service host, regardless of its packaging and environment, is a **node**. You can have multiple service host instances, each with their own ports, within a single process, all isolated from each other, or within the same container, same VM, bare metal OS etc. For the purposes of the node group membership each one is treated as a independent node.

## Node Options

A node can participate in a group as a **peer** or as an **observer**. Peer is the default behavior and when a node group is created, it adds a NodeState instance to represent itself in the group, as a peer.

An observer node is a node that can be a member of a group, receive gossip updates on group membership changes, but is not eligible to receive replication, broadcast or forwarded requests. Its simple a consumer/client of services running on all the other nodes in the group. To become a peer, when the node group is created locally, it might have an initial state with the option set to OBSERVER.

# REST API

## URI
```
/core/node-groups/default
/core/node-groups/some-group-name

```


## GET
```json
curl http://192.168.1.33:8000/core/node-groups/default
{
  "documents": {
    "/core/node-groups/default/130fad85-70fa-4e9a-848d-78b3c025a425": {
      "groupReference": "http://192.168.1.33:8000/core/node-groups/default",
      "status": "AVAILABLE",
      "options": "PEER, SYNCHRONIZE_ON_JOIN",
      "id": "130fad85-70fa-4e9a-848d-78b3c025a425",
      "documentVersion": 0,
      "documentKind": "com:vmware:dcp:services:common:NodeState",
      "documentSelfLink": "/core/node-groups/default/130fad85-70fa-4e9a-848d-78b3c025a425",
      "documentUpdateTimeMicros": 1415321955360000,
      "documentExpirationTimeMicros": 0
    },
    "/core/node-groups/default/2256569e-c5f3-4a93-a593-ac355b610a33": {
      "groupReference": "http://192.168.1.98:8000/core/node-groups/default", 
      "status": "AVAILABLE",
      "options": "PEER, SYNCHRONIZE_ON_JOIN",
      "id": "2256569e-c5f3-4a93-a593-ac355b610a33",
      "documentVersion": 0,
      "documentKind": "com:vmware:dcp:services:common:NodeState",
      "documentSelfLink": "/core/node-groups/default/2256569e-c5f3-4a93-a593-ac355b610a33",
      "documentUpdateTimeMicros": 1415321955328000,
      "documentExpirationTimeMicros": 0
    },
}
```

## POST

A Xenon node joins an existing group of nodes, by sending a POST to its local /core/node-groups/some-group service, with the following body:

```
    public static class JoinPeerRequest {
        /**
         * Existing member of the group we wish to join through
         */
        public URI memberGroupReference;

        /**
         * Minimum number of nodes to enumeration, after join, for synchronization to start
         */
        public Integer synchQuorum;
    }

```


## Node state

Each node in the group is represented with the NodeState PODO
```
public class NodeState extends ServiceDocument {
    public enum NodeStatus {
        /**
         * Node status is unknown
         */
        UNKNOWN,

        /**
         * Node is not available and is not responding to membership update requests
         */
        UNAVAILABLE,

        /**
         * Node is marked as healthy, it has successfully reported membership during the last update
         * interval
         */
        AVAILABLE,

        /**
         * Node is healthy but synchronizing state with its peers.
         */
        SYNCHRONIZING,

        /**
         * Node has been replaced by a new node, listening on the same IP address and port and is no
         * longer active
         */
        REPLACED,

        /**
         * Node has initiated local replicated service restart and will transition to available when local service synchronize state with peer replicas
         */
        RESTARTING_SERVICES,
    }

    public URI groupReference;
    public NodeStatus status = NodeStatus.UNKNOWN;
    public String id;

    /**
     * Minimum number of available nodes required to accept a proposal for replicated updates to
     * succeed
     */
    public int membershipQuorum = 1;
}
```



