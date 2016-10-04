# Overview
The node group service tracks a group of Xenon nodes. It does failure detection and membership update between all nodes using an implementation of the SWIM protocol. 

The role of this service is described in the [design page](./Implementors-Guide) and [replication protocol page](./Leader-Election-And-Replication-Design)

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

### Utility URIs

#### Available
```
/core/node-groups/some-group/available
```

GET to the /available suffix will return 200 (OK) if the node group meets the following criteria
 * There have been no updates to node state through gossip for a certain interval (configurable through /config)
 * The node group has at least N healthy members, where N is greater or equal to the greatest between membership forum and synchronization forum

Otherwise, when the group is in flux, or quorum of nodes is not met, it will return 503 (UNAVAILABLE).

#### Stats
```
/core/node-groups/some-group/stats
```
Several important stats including gossip attempts are tracked in real time by the node group service

## GET

`GET http://127.0.0.1:63742/core/node-groups/default'
```json
  "config": {
    "nodeRemovalDelayMicros": 3600000000,
    "stableGroupMaintenanceIntervalCount": 5
  },
  "nodes": {
    "host-2": {
      "groupReference": "http://127.0.0.1:63742/core/node-groups/default",
      "status": "AVAILABLE",
      "options": [
        "PEER"
      ],
      "id": "host-2",
      "membershipQuorum": 1,
      "synchQuorum": 2,
      "documentVersion": 3,
      "documentKind": "com:vmware:xenon:services:common:NodeState",
      "documentSelfLink": "/core/node-groups/default/host-2",
      "documentUpdateTimeMicros": 1454693047110000,
      "documentExpirationTimeMicros": 0
    },
    "host-3": {
      "groupReference": "http://127.0.0.1:63738/core/node-groups/default",
      "status": "AVAILABLE",
      "options": [
        "PEER"
      ],
      "id": "host-3",
      "membershipQuorum": 1,
      "synchQuorum": 2,
      "documentVersion": 3,
      "documentKind": "com:vmware:xenon:services:common:NodeState",
      "documentSelfLink": "/core/node-groups/default/host-3",
      "documentUpdateTimeMicros": 1454693063410098,
      "documentExpirationTimeMicros": 0
    }
  },
  "membershipUpdateTimeMicros": 1454693047849007,
  "documentVersion": 377,
  "documentKind": "com:vmware:xenon:services:common:NodeGroupService:NodeGroupState",
  "documentSelfLink": "/core/node-groups/default",
  "documentUpdateTimeMicros": 1454693064466000,
  "documentUpdateAction": "PATCH",
  "documentExpirationTimeMicros": 0,
  "documentOwner": "host-2"
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

# Node-Group Convergence
When a node-group goes through changes i.e. node(s) get added or removed, the
gossip protocol as it learns about these changes, may generate a lot of node-group
change events. This is due to the nature of the protocol, as nodes learn and 
propagate their information through peers. One interesting challenge that arises 
is to determine reliably when the node-group has converged i.e. all nodes in the
node-group are in sync and reflect the same view of the node-group. This is 
important because, many critical tasks in Xenon are dependent on node-group 
changes, for example Synchronization. 

To determine node-group convergence, Xenon uses the property membershipUpdateTimeMicros
in the NodeGroupState PODO. This property reflects the maximum value among all 
reported update times from peer nodes. As gossip propagates information about each
peer to remaining nodes, this property settles to the same value across all peers.
So, to check for node-group convergence, xenon queries the node-group service on
each peer to check if everyone reports the same value for membershipUpdateTimeMicros.
If they do, the node-group is considered converged. 

Node-Group convergence is driven by the associated Node-Selector, which receives
node-group change notifications. The node-selector checks for convergence on these
updates and when the node-group has converged, notifies the Xenon host. For 
reference, see method NodeGroupUtils.checkConvergence that is used by the 
Node-Selector to verify convergence.
