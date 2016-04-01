# Overview
This tutorial covers topics relating to multi node deployments, and the concepts behind high availability, leader election, and service restart during node group changes. It demonstrates the ease of developing robust, scale out workflows, modeled as task services, and the service options, best practices needed.

# Prerequisites

While replicated services deployed across multiple nodes should be the default type of service, this is an advanced topic since it requires some understanding of distributed system concepts, and the xenon programming model. Please make sure to read the tutorials below and *their* prerequisites

* [Example Tutorial](./Example-Service-Tutorial)
* [Multi Node Tutorial](./Multi-Node-Tutorial)
* [Xenon Clustering deck](https://github.com/vmware/xenon/blob/master/contrib/docs/XenonClustering.pptx)

# Key Concepts

Xenon relies on two mechanisms to understand and properly implement the needs of a service:

 * Service options - ServiceOption.OWNER_SELECTION and ServiceOption.REPLICATION
 * Node group configuration - Membership quorum

Service instances run on each node but the consensus protocol routes requests to a single node, the one elected as owner, for a particular service.

Using the same underlying protocol the runtime will guarantee that
 * updates happen atomically, service handlers execute only on service instance on owner node
 * periodic handleMaintenance only executes on owner node
 * handleStart only executes on owner node

The service author does not need to know which node is owner. It simply authors logic assuming it only runs on one node, and if nodes come and go, a new node will dynamically take over.

leader election and high availability come "embedded" in the programming model: since the update, maintenance and start handlers are guaranteed to run on one node, out of N, the author can initiate logic assuming the peers will see state changes and persist them, but will not duplicate the effort.

#Demonstration

[TaskService](https://github.com/vmware/xenon/wiki/Task-Service-Tutorial) is kind of service which contains a series of tasks and has its own workflow. Here we use the [ExampleTaskService ](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/ExampleTaskService.java)which will query and delete all example service documents to demonstrate the scale out, and restart, across nodes. [Task Service Tutorial](https://github.com/vmware/xenon/wiki/Task-Service-Tutorial) tells more about the ExampleTaskService.

##1. Start a node group

We are going to start the [ExampleServiceHost](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/ExampleServiceHost.java), which will start the factory service for creating ExampleTaskService. We use three ports: 8000, 8001, 8002 for testing, so open three terminals and use the commands to start each node:

```
% java -Dxenon.NodeState.membershipQuorum=3 -cp xenon-host/target/xenon-host-0.7.6-SNAPSHOT-jar-with-dependencies.jar com.vmware.xenon.services.common.ExampleServiceHost --sandbox=/tmp/xenon --port=8000 --peerNodes=http://127.0.0.1:8000,http://127.0.0.1:8001,http://127.0.0.1:8002 --id=hostAtPort8000 

% java -Dxenon.NodeState.membershipQuorum=3 -cp xenon-host/target/xenon-host-0.7.6-SNAPSHOT-jar-with-dependencies.jar com.vmware.xenon.services.common.ExampleServiceHost --sandbox=/tmp/xenon --port=8001 --peerNodes=http://127.0.0.1:8000,http://127.0.0.1:8001,http://127.0.0.1:8002 --id=hostAtPort8001

% java -Dxenon.NodeState.membershipQuorum=3 -cp xenon-host/target/xenon-host-0.7.6-SNAPSHOT-jar-with-dependencies.jar com.vmware.xenon.services.common.ExampleServiceHost --sandbox=/tmp/xenon --port=8002 --peerNodes=http://127.0.0.1:8000,http://127.0.0.1:8001,http://127.0.0.1:8002 --id=hostAtPort8002
```

Here we use --id=hostAtPort8000 to specify an ID of the host, and use --peerNodes to specify the group members. Also, we specify the membershipQuorum by using a JVM xenon property "-Dxenon.NodeState.membershipQuorum=3". [membershipQuorum](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/NodeState.java) is the minimum number of available nodes required for consensus operations and synchronization, since we need to wait for all nodes to be available when starting a cluster, it's safest to set it to total number of nodes.

When joining to a group, each node will send a JoinPeerRequest to the peers. Then each peer handles the join post and sets its own membershipQuorum as the maximum of its pre-specified membershipQuorum and other's join request's membershipQuorum:
```Java
self.membershipQuorum = Math.max(self.membershipQuorum, joinBody.membershipQuorum);
```
Now we can see the node group has been created:

```
% curl http://localhost:8000/core/node-groups/default

{
  ...
  "nodes": {
    "hostAtPort8000": {
      "status": "AVAILABLE",
      "membershipQuorum": 3,
      ...
    },
    "hostAtPort8001": {
      "status": "AVAILABLE",
      "id": "hostAtPort8001",
      "membershipQuorum": 3,
      ...
    },
    "hostAtPort8002": {
      "status": "AVAILABLE",
      "id": "hostAtPort8002",
      "membershipQuorum": 3,
      ...
    }
  },
  "documentOwner": "hostAtPort8000",
  ...
}
```

##2. Start ExampleTaskService 

At first, start a example service, which will be deleted by the ExampleTaskService later.

_File:_ example-1.body
```
{
  "name": "example-1",
  "counter": 1
}
```

```
% curl -s -X POST -d@example-1.body http://localhost:8000/core/examples -H "Content-Type: application/json"

{
  "documentLinks": [
    "/core/examples/e6691e79-a9ff-48eb-bd96-953e33e2fc1b"
  ],
  "documentCount": 1,
  "queryTimeMicros": 6999,
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "hostAtPort8000"
}
```

Then, create two example task services:

_File:_ task.body
```
{
  "taskLifetime": 300
}
```

ExampleTaskService-1:

```
% curl -s -X POST -d@task.body http://localhost:8000/core/example-tasks -H "Content-Type: application/json"

{
  "taskLifetime": 300,
  "subStage": "QUERY_EXAMPLES",
  "taskInfo": {
    "stage": "STARTED",
    "isDirect": false
  },
  "documentVersion": 0,
  "documentSelfLink": "/core/example-tasks/97b284e1-5b81-46b6-93c0-181ea346fb2d",
  "documentOwner": "hostAtPort8001"
  ...
}
```

ExampleTaskService-2:

```
% curl -s -X POST -d@task.body http://localhost:8000/core/example-tasks -H "Content-Type: application/json"

{
  "taskLifetime": 300,
  "subStage": "QUERY_EXAMPLES",
  "taskInfo": {
    "stage": "STARTED",
    "isDirect": false
  },
  "documentVersion": 0,
  "documentSelfLink": "/core/example-tasks/538d8398-0ca1-4af2-b5f6-9be30da56081",
  "documentOwner": "hostAtPort8000"
  ...
}
```

We can see that the two task services has been started.

##3. ExampleTaskService is working

In order to show how they go through their stages, we create a query task with INCLUDE_ALL_VERSIONS:

_File:_ query.body

```
{
    "taskInfo": {
        "isDirect": true
    },
    "querySpec": {
        "options": [
            "INCLUDE_ALL_VERSIONS"
        ],
        "query": {
            "term": {
                "matchValue": "com:vmware:xenon:services:common:ExampleTaskService:ExampleTaskServiceState",
                "propertyName": "documentKind"
            }
        }
    }
}
```

```
% curl -s -X POST -d@query.body http://localhost:8000/core/query-tasks -H "Content-Type: application/json"

{
  "taskInfo": {
    "stage": "FINISHED",
    "isDirect": true,
    "durationMicros": 994
  },
  "querySpec": {
    "query": {
      "occurance": "MUST_OCCUR",
      "term": {
        "propertyName": "documentKind",
        "matchValue": "com:vmware:xenon:services:common:ExampleTaskService:ExampleTaskServiceState",
        "matchType": "TERM"
      }
    },
    "resultLimit": 2147483647,
    "options": [
      "INCLUDE_ALL_VERSIONS"
    ]
  },
  "results": {
    "documentLinks": [
      "/core/example-tasks/97b284e1-5b81-46b6-93c0-181ea346fb2d?documentVersion=3",
      "/core/example-tasks/538d8398-0ca1-4af2-b5f6-9be30da56081?documentVersion=3",
      "/core/example-tasks/97b284e1-5b81-46b6-93c0-181ea346fb2d?documentVersion=2",
      "/core/example-tasks/538d8398-0ca1-4af2-b5f6-9be30da56081?documentVersion=2",
      "/core/example-tasks/97b284e1-5b81-46b6-93c0-181ea346fb2d?documentVersion=1",
      "/core/example-tasks/538d8398-0ca1-4af2-b5f6-9be30da56081?documentVersion=1",
      "/core/example-tasks/97b284e1-5b81-46b6-93c0-181ea346fb2d?documentVersion=0",
      "/core/example-tasks/538d8398-0ca1-4af2-b5f6-9be30da56081?documentVersion=0"
    ],
  ...
    "documentOwner": "hostAtPort8001"
  },
  "documentOwner": "hostAtPort8001",
  ...
}
```

We can see each ExampleTaskService's documentVersion changes from 0 - 3, since when each stage finished, it will send a self-patch to update the documentVersion.

Now the example service has been deleted by the task:

```
% curl http://localhost:8000/core/examples/

{
  "documentLinks": [],
  "documentCount": 0,
  "queryTimeMicros": 998,
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "hostAtPort8000"
}
```

##4. Stop one node

Either with Ctrl-C, or better, sending a DELETE to /core/management on one of the nodes, here we delete node: 8002.

```
curl -X DELETE http://localhost:8002/core/management
```

##5. Create a new task service fails

_File:_ task.body

```
{
  "taskLifetime": 300
}
```

```
% curl -s -X POST -d@task.body http://localhost:8000/core/example-tasks -H "Content-Type: application/json"
```

The treminal shows no response and the process is blocked, because quorum is still set to 3 and only 2 nodes remain, consensus operations and synchronization can't be executed. Creating this new task service fails, so just Ctrl-C to exit.

##6. Relax the quorum

Now relax the quorum by sending a UpdateQuorumRequest PATCH to one of the remaining nodes and set membershipQuorum = 2.

_File:_ updateQuorum.body

```
{
    "isGroupUpdate" : true,
    "membershipQuorum" : 2,
    "kind" : "com:vmware:xenon:services:common:NodeGroupService:UpdateQuorumRequest"
}
```

```
% curl -X PATCH -d@updateQuorum.body http://localhost:8000/core/node-groups/default -H "Content-type: application/json"
```

Now we can see node 8000 and 8001's membershipQuorum is 2 and 8002's is still 3.

##7. New task service created

And we also notice that the previous failure request (create a new task service) works automatically. New task service (/core/example-tasks/b7d49734-903e-4b24-9326-0125866ccefb) has been created: 

```
% curl http://localhost:8000/core/example-tasks

{
  "documentLinks": [
    "/core/example-tasks/97b284e1-5b81-46b6-93c0-181ea346fb2d",
    "/core/example-tasks/538d8398-0ca1-4af2-b5f6-9be30da56081",
    "/core/example-tasks/b7d49734-903e-4b24-9326-0125866ccefb"
  ],
  "documentCount": 3,
  "queryTimeMicros": 1997,
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "hostAtPort8000"
}
```

Let's see the logs to explore how synchronization occurs automatically:

```
Node 8000's logs:
[120][I][1459374473536][8000/core/example-tasks/b7d49734-903e-4b24-9326-0125866ccefb][handleSynchronizeWithPeersCompletion][isOwner:false e:0 v:3, cause:null]

Node 8001's logs:
[128][I][1459374472611][8001/core/example-tasks/b7d49734-903e-4b24-9326-0125866ccefb][handleSynchronizeWithPeersCompletion][isOwner:true e:0 v:3, cause:java.lang.IllegalStateException: PATCH to /core/example-tasks/b7d49734-903e-4b24-9326-0125866ccefb failed. Success: 1,  Fail: 1, quorum: 2, threshold: 1]
```

We can see the new task service (b7d49734-903e-4b24-9326-0125866ccefb)'s onwer is 8001, and it has been synchronized to node 8000. Also, "Success: 1,  Fail: 1" of 8001's log means synchronization is successful with node 8000 but failure with node 8002, since it's closed.

##8. Restart the node you stopped

Now let's restart the node 8002 and set the membershipQuorum=2:

```
% java -Dxenon.NodeState.membershipQuorum=2 -cp xenon-host/target/xenon-host-0.7.6-SNAPSHOT-jar-with-dependencies.jar com.vmware.xenon.services.common.ExampleServiceHost --sandbox=/tmp/xenon --port=8002 --peerNodes=http://127.0.0.1:8000,http://127.0.0.1:8001,http://127.0.0.1:8002 --id=hostAtPort8002
```

```
% curl http://localhost:8000/core/node-groups/default
```

You'll see the node 8002's status is AVAILABLE.

##9. Demonstrate synchronization

Send a GET to the example task factory of the restarted node, that all the tasks are there:

```
% curl http://localhost:8002/core/example-tasks

{
  "documentLinks": [
    "/core/example-tasks/97b284e1-5b81-46b6-93c0-181ea346fb2d",
    "/core/example-tasks/538d8398-0ca1-4af2-b5f6-9be30da56081",
    "/core/example-tasks/b7d49734-903e-4b24-9326-0125866ccefb"
  ],
  "documentCount": 3,
  "queryTimeMicros": 1997,
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "hostAtPort8002"
}
```

And the same with node 8000, 8001.

Check each task service's status, they are all finished.

For example:

```
% curl http://localhost:8002/core/example-tasks/b7d49734-903e-4b24-9326-0125866ccefb

{
  "taskLifetime": 300,
  "exampleQueryTask": {
    "taskInfo": {
      "stage": "FINISHED",
      "isDirect": true,
      "durationMicros": 999
    },
    ...
  },
  "taskInfo": {
    "stage": "FINISHED",
    "isDirect": false
  },
  "documentSelfLink": "/core/example-tasks/b7d49734-903e-4b24-9326-0125866ccefb",
  "documentOwner": "hostAtPort8001",
  ...
}
```
