# Overview
The node selector synchronization service is associated with an instance of a [node selector service](nodeSelectorService) and is invoked to synchronize document versions and document epoch between replicated service instances.

The synchronization process is invoked when a node is determined to be the new owner of a service, either on startup or due to node group changes noticed during the steady state replication protocol.

Please refer to the [replication protocol page](leaderElectionAndReplicationDesignPage) for the overall replication and consensus scheme

# REST API

## URI
```
/core/node-selectors/default/synch
/core/node-selectors/<customname>/synch
```

## POST
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

# Synchronization triggers

State synchronization can be triggered on demand or automatically when a node selector service notices node group changes:

* Client issues PATCH to /core/management with a SynchronizeWithPeers request
* Service code calls serviceHost.scheduleNodeGroupChangeMaintenance()
* A replicated, owner selected service notices that the documentOwner or documentEpoch in the request body does not match its calculation for owner and epoch. It will trigger synchronization with peers and then fail the original request, which will be retried by the node that received the client request

Factory services issues synchronization requests in batches, for their locally enumerated children. A factory service kicks of synchronization when it receives a maintenance operation with MaintenanceReason.NODE_GROUP_CHANGE

## Synchronization steps

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



