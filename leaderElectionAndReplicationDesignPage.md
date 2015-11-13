# Overview

Xenon  separates  the  processes  of  node  group  maintenance,  selection
of  a primary  (leader,  owner)  node per  service,  and  that of  state
update replication.  Xenon allows  a service  author to  declare a  set of
capabilities  on the  service, which  the framework  uses at  runtime to
employ the proper protocol and processing.

## Required reading

The  content that  follows will  make  a lot  more sense  if the  reader
reviews the following pages. Its  especially recommended the reader goes
through the example tutorial

 * [Glossary](Glossary)
 * [Programming model](dcp-Programming-Model)

Please refer to the [design page](dcp-Design) for larger context and Xenon
motivation.

## Service Options

Xenon offers different guarantees to  a service instance, depending on the
service options  declared in the  service class. Please  see programming
model for details. This page relates to the following two options:

 * ServiceOption.OWNER_SELECTION
 * ServiceOption.EAGER_CONSISTENCY

Both of the above options _require_ ServiceOption.REPLICATION

### CAP Spectrum using Service Options

Xenon provides three levels of consistency+availability, expressed through
the combination of Capabilities, **per service instance**.

#### Level 1 ServiceOption.REPLICATION

No consistency  guarantees, highest  availability. Updates to  a service
instance are applied  on the node they are received  and then replicated
to peers if the update is  validated and locally committed. Xenon will not
wait for peers to respond before completing request to client.

#### Level 2 ServiceOption.REPLICATION + ServiceOption.OWNER_SELECTION 

This is the primary scale  out mechanism. Higher consistency, allows for
group membership changes while updates  occur. All updates are forwarded
to  service  instance  on  node assigned  as  "owner",  assuming  stable
membership.  This  follows the  strong  leader  principle, allowing  the
owner, per service,  to see all updates first, then  replicate if update
is  valid.  Since updates  are  serialized  within a  service  instance,
clients will observe ordered updates on the owner node.

This  level, and  the  next  use (i)  a  _consensus  protocol_ and  (ii)
_synchronous_ replication: the client does  not see a response until the
consensus protocol executes.

The handleStart and handle PATCH/PUT/DELETE/GET handlers only get
invoked on the service instance running on the owner node.

#### Level 3 ServiceOption.REPLICATION + ServiceOption.OWNER_SELECTION + ServiceOption.EAGER_CONSISTENCY

This combination is identical to level 2, with one crucial difference
: if the number of available nodes is below the quorum level, the
request will fail.

## Ordering

Requests are only guaranteed to be processed in the order they arrive at
the selected  “owner” (leader)  node per service.  If the  _same_ client
does not  wait for completion  of a  request before sending  another, no
guarantee is made  on what order the client will  see the completions to
each request.

The service concurrency management guarantees  that only one update will
be  processed, on  the same  node and  the same  service instance.  This
guarantee, coupled with the owner  selection and automatic forwarding of
all requests  for the same service  path, to the same  node, means that,
requests will evolve the state  version of a service, deterministically.
Possible  divergence  occurs  when  two clients  get  forwarded  to  two
different selected owners for the same service path, and they have a non
intersecting view of  the membership, due to  gossip replication delays.
The probability of this occurring  is minimized by not forwarding client
requests when membership is in flux.

## Duplicate requests and Idempotent Behavior

Unlike RAFT and VR we do not expect the Xenon framework to track duplicate
requests.  The notion  of "duplicate"  in RAFT  is simplistic  since two
different clients can attempt to  issue a semantically duplicate request
(causes the same work to be done by the service). Detecting duplicate or
out of date work can happen in one of two ways:

 *  The   service  has   ServiceOption.STRICT_UPDATE_CHECKING  requiring
 conditional updates. This  means the client must have  read the current
 state, from  the owner node, and  supply the same version  and document
 signature  for the  update to  be accepted.  In this  case, client  and
 framework ensure idempotency

 * The service  author adds logic in the PATCH,  PUT message handlers to
 detect when  a request attempts  to move state "backwards"  or requests
 duplicate work. This can be a simple check of some request field values
 against  the  state,  essentially  implementing  a  custom  version  of
 ServiceOption.STRICT_UPDATE_CHECKING

## Conflict / Error detection / Divergence

If  the service  is marked  with OWNER_SELECTION  and an  update is  not
accepted by a majority of peers, the client sees failures and the update
is not committed at any service  instance (owner or peers). For services
that do  not require eager consistency,  the update is committed  on the
owner and all replicas that accepted it but a service statistic is added
to each service instance marking the service as IN-CONFLICT.

Divergence  of values  between replicas  can occur  for several  reasons
(including network  partitioning) so we  don't believe a system  can get
away without a  eventual consistency mechanism that  periodically, or on
conflict  detection attempts  to converge  values (through  anti-entropy
process).  So eventual  consistency is  not exclusive  of eager,  strong
consistency  and  provides the  same  simple  programming model  to  the
service author

# Protocol details

## Comparison

The approach outlined below is  most similar to view stamped replication
(VR).

 * [Raft analysis](http://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-857.pdf)
 * [Viewstamped Replication Revisited](http://pmg.csail.mit.edu/papers/vr-revisited.pdf)
 * [Comparison of Paxos, VR, Zookeeper Atomic Broadcast] (https://www.cs.cornell.edu/fbs/publications/viveLaDifference.pdf)

Due  to  the   Xenon  service-as-a-document  model,  and  our   use  of  a
multi-version  index, the  following Xenon  fields and  mechanisms map  to
concepts in VR / Raft:

#### Equivalent concepts

 *  Each update  results in  the service  state being  updated. This  is
 essentially a snapshot, indexed as a new version, and allowing recovery
 to  proceed with  just the  latest  version, not  requiring the  entire
 history of updates. The log in our  case is a history of complete state
 snapshots

 * The  view, or term, is  the documentOwner field, plus the 
documentEpoch, which are present in every version of the state.

 *  The commit  number  is the  documentVersion  field, which  increases
 monotonically,  at   the  selected  owner  node,   on  each  successful
 replication with majority of peers.

#### Differences

 * Owner selection (or leader election)  is done using a consistent hash
 of  the  service  URI  path,  over  the  node  membership.  Since  node
 membership is maintained by gossip,  divergent views are possible. That
 is ok, since  we require the majority  to agree on the  owner, for each
 update.

 * Owner  selection happens per request,  since we hash the  service URI
 path to  the view of the  nodes, at each  peer. There is no  voting, no
 timeout.  As part  of  replication  or request  processing,  if a  node
 notices it  does not  have the  same owner selected, or disagrees with
 the epoch,  it will  fail the request and re-synchronization will occur

## Required Service Options

The following service options must be enabled on a service for the following protocol to take effect, on every request update:
 * PERSISTANCE
 * REPLICATION
 * OWNER_SELECTION 

The EAGER_CONSISTENCY option determines if a request is committed only when group membership is stable and a majority of nodes accept the update. 

## Leader/Owner Selection (View Progression/Reconfiguration)

Each Xenon node belongs to one or more node groups. Each node group is maintained by an instance of the [node group service](NodeGroupService) which implements random gossip, see [SWIM](http://www.cs.cornell.edu/~asdas/research/dsn02-swim.pdf).

Xenon, using the default node selector service, uses consistent hashing to assign a key (for this discussion a service instance link), to a Xenon node. 

### Steady state
The ConsistentHashingNodeSelectorService performs the following steps, per client request
 * Retrieves the contents of the node group - This is a group of service hosts that is kept current through gossip. It uses the same underlying algorithm as peer to peer algorithms, detailed in the SWIM paper. 
 * Hashes a key (the service instance URI path, or a key) to one of the node ids. We take the euclidean distance between the SHA256 hash of the key, and the SHA256 hash of each node id. The node id with the smallest distance to the key, is designated the owner. If the group membership changed within a maintenance interval the results are returned immediately. Otherwise the partitioning request is queued and attempted on next maintenance interval or until expiration. 
 * The replication state machine, which runs as part of the I/O pipeline, post operation completion on the owner replica, sends a validated state to all secondary replicas of the service
 * Each replica validates:
  * The owner id (nodes must agree on who the owner is)
  * The document epoch (this is a number incremented when a new owner is selected)

## Synchronization State Machine (Reconfiguration)

Synchronization occurs when new nodes are added to a node group, or existing nodes become un-available. Synchronization can be triggered through a client request to the core management service (PATCH to /core/management with a specific body) or it can be triggered from a notification on a node group.

The steady state protocol outlined below will also trigger a per service resynchronization if it notices disagreement on epoch, owner or version.

Please see  the  [synchronization page](nodeGroupSynchronizationService).


## Replication protocol
(Implemented by service client and service host inbound queuing logic)

 * Request from client is forwarded to owner/leader node. If on the same node, request is queued and dispatched to service instance
 * If request was forwarded and node fails to respond (fails with I/O error to request times out):
  * Queue failed request
  * Issue gossip patch to update node membership
  * When group membership is stable, queued request is retried (new owner should have been selected)
 * Service on owner node receives request:
 * If STRICT_UPDATE_CHECKING required, performs comparison (See conditional update state machine)
 * Service handler executes
 * If Service handler fails request, STOP (client sees failure)
 * On request completion by service instance:
  * Propose: Owner issues a PUT, with the updated state to the peers. Body specifies documentOwner (the view), documentEpoch, and documentVersion (the current state version). 
   * Each peer verifies they agree on the owner
   * Each peer verifies they agree on the current version of proposal
   * Each peer verifies they agree on the epoch
   * If peer agrees on version,epoch and owner, it replies with success to the PUT. The state is indexed using the proposed version
   * If peer does not agree with owner or version, fails request
   * If majority of peers fail request, STOP, client(sees failure). Peers garbage collect any un committed state
  * Learn: After collecting majority responses from peers
   * Owner node indexes request locally. If indexing fails, client sees failure.
   * Owner completes forwarded request from entry point node
  * Decide (commit): If no more requests are pending, owner node sends the current (committed) state to all peers, serving as "commit" to all the pending proposals at the peers. We do this only when the owner queue is empty to minimize additional messaging cost.
 * Entry point node (which forwarded to owner), completes request to client


