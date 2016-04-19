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

[TaskService](https://github.com/vmware/xenon/wiki/Task-Service-Tutorial) is a kind of service which contains a series of tasks and has its own workflow. Here we use the [ExampleTaskService ](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/ExampleTaskService.java)which will query and delete all example service documents to demonstrate the scale out, and restart, across nodes. [Task Service Tutorial](https://github.com/vmware/xenon/wiki/Task-Service-Tutorial) tells more about the ExampleTaskService.

Here are the main steps for the demonstration:

1. Start a node group.
2. Start ExampleTaskService.
3. Examine ExampleTaskService.
4. Stop one node.
5. Verify the quorum is enforced.
6. Relax the quorum.
7. Verify task creation with relaxed quorum.
8. Restart node.
9. Verify synchronization.

##1 Start a node group

We use three ports: 8000, 8001, 8002 for testing, so open three terminals and use the commands to start each node:

```
% java -Dxenon.NodeState.membershipQuorum=3 -cp xenon-host/target/xenon-host-0.7.6-SNAPSHOT-jar-with-dependencies.jar com.vmware.xenon.services.common.ExampleServiceHost --sandbox=/tmp/xenon --port=8000 --peerNodes=http://127.0.0.1:8000,http://127.0.0.1:8001,http://127.0.0.1:8002 --id=hostAtPort8000 

% java -Dxenon.NodeState.membershipQuorum=3 -cp xenon-host/target/xenon-host-0.7.6-SNAPSHOT-jar-with-dependencies.jar com.vmware.xenon.services.common.ExampleServiceHost --sandbox=/tmp/xenon --port=8001 --peerNodes=http://127.0.0.1:8000,http://127.0.0.1:8001,http://127.0.0.1:8002 --id=hostAtPort8001

% java -Dxenon.NodeState.membershipQuorum=3 -cp xenon-host/target/xenon-host-0.7.6-SNAPSHOT-jar-with-dependencies.jar com.vmware.xenon.services.common.ExampleServiceHost --sandbox=/tmp/xenon --port=8002 --peerNodes=http://127.0.0.1:8000,http://127.0.0.1:8001,http://127.0.0.1:8002 --id=hostAtPort8002
```

Here are the explanations for each argument.

###1.1 Specify membership quorum 

[membershipQuorum](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/NodeState.java) is the minimum number of available nodes required for consensus operations and synchronization, we can specify the membershipQuorum by using a JVM xenon property "-Dxenon.NodeState.membershipQuorum=3". Since we need to wait for all nodes to be available when starting a cluster, it's the safest to set it to total number of nodes.

###1.2 Specify service host

We can specify the class path "com.vmware.xenon.services.common.ExampleServiceHost" to start the [ExampleServiceHost](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/ExampleServiceHost.java), which will start the factory service for creating ExampleTaskService. 

###1.3 Specify sandbox

We are going to provide an argument for the sandbox (where all service documents are stored), instead of relying on the Java temporary directory. When the host is not running, you can delete the sandbox if you need to get back to "factory settings".

###1.4 Specify peer nodes

Use "--peerNodes" to specify the group members. You can either dynamically join a node group by sending a POST to the local (un-joined) node group service at /core/node-groups/default, or, you can have the xenon host join when it starts, by supplying the --peerNodes argument, with a list of URIs. To make startup scripts simple, you can supply the **same set of URIs, including the IP:PORT of the local host**. Xenon will ignore its self address but concurrently join through the other peer addresses. For more details on node group maintenance, see the [node group service](https://github.com/vmware/xenon/wiki/NodeGroupService)

###1.5 Specify host ID

Instead of hard to remember class 4 UUIDs, you can specify an ID when you start a Xenon host by providing the argument like "--id=hostAtPort8000".

###1.6 Show group status

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
  ...
}
```

##2 Start ExampleTaskService  

### 2.1 Start example service 

At first, start an example service, which will be deleted by the ExampleTaskService later.

_File:_ example-1.body
```
{
  "name": "example-1"
}
```

```
% curl -s -X POST -d@example-1.body http://localhost:8000/core/examples -H "Content-Type: application/json"

{
  "documentLinks": [
    "/core/examples/e6691e79-a9ff-48eb-bd96-953e33e2fc1b"
  ],
  "documentOwner": "hostAtPort8000",
  ...
}
```

### 2.2 Create the first task service

Here we specify the life time of the task to 600 seconds for testing. If you want the task to live longer in your test, you can extend it.

_File:_ task-1.body
```
{
  "taskLifetime": 600,
  "documentSelfLink":"task-1"
}
```

ExampleTaskService-1:

```
% curl -s -X POST -d@task-1.body http://localhost:8000/core/example-tasks -H "Content-Type: application/json"

{
  "taskLifetime": 600,
  "subStage": "QUERY_EXAMPLES",
  "taskInfo": {
    "stage": "STARTED",
    "isDirect": false
  },
  "documentVersion": 0,
  "documentSelfLink": "/core/example-tasks/task-1",
  "documentOwner": "hostAtPort8001"
  ...
}
```

### 2.3 Create the second task service

_File:_ task-2.body

```
{
  "taskLifetime": 600,
  "documentSelfLink":"task-2"
}
```

ExampleTaskService-2:

```
% curl -s -X POST -d@task-2.body http://localhost:8000/core/example-tasks -H "Content-Type: application/json"

{
  "taskLifetime": 600,
  "subStage": "QUERY_EXAMPLES",
  "taskInfo": {
    "stage": "STARTED",
    "isDirect": false
  },
  "documentVersion": 0,
  "documentSelfLink": "/core/example-tasks/task-2",
  "documentOwner": "hostAtPort8000"
  ...
}
```

We can see that the two task services has been started.

##3. Examine ExampleTaskService

In order to show how they go through their stages, we create a query task with INCLUDE_ALL_VERSIONS, then the query will execute over all document versions, not just the latest per self link:

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
      "/core/example-tasks/task-1?documentVersion=3",
      "/core/example-tasks/task-2?documentVersion=3",
      "/core/example-tasks/task-1?documentVersion=2",
      "/core/example-tasks/task-2?documentVersion=2",
      "/core/example-tasks/task-1?documentVersion=1",
      "/core/example-tasks/task-2?documentVersion=1",
      "/core/example-tasks/task-1?documentVersion=0",
      "/core/example-tasks/task-2?documentVersion=0"
    ],
  ...
    "documentOwner": "hostAtPort8001"
  },
  ...
}
```

We can see each ExampleTaskService's documentVersion changes from 0 to 3, since when each stage finished, it will send a self-patch to update the documentVersion.

Since the main job of [ExampleTaskService](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/ExampleTaskService.java) is to query and delete the example services, now the example service has been deleted by the task services:

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
% curl -X DELETE http://localhost:8002/core/management
```

##5. Verify the quorum is enforced

In order to verify the quorum is enforced, create another task service to see whether it can be created successfully when one node has been stopped.

_File:_ task-3.body

```
{
  "taskLifetime": 600,
  "documentSelfLink":"task-3"
}
```

```
% curl -s -X POST -d@task-3.body http://localhost:8000/core/example-tasks -H "Content-Type: application/json"
```

The terminal shows no response and the process is blocked, because quorum is still set to 3 and only 2 nodes remain, consensus operations and synchronization can't be executed. Creating this new task service fails, so just Ctrl-C to exit.

##6. Relax the quorum

Now relax the quorum by sending a UpdateQuorumRequest PATCH to the [NodeGroupService](https://github.com/vmware/xenon/wiki/NodeGroupService) of one remaining node and set membershipQuorum = 2.

When "isGroupUpdate" is set true, the PATCH will act on the whole group.

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

##7. Verify task creation with relaxed quorum

And we also notice that the previous failure request (create a new task service) works automatically. The service "task-3" has been created: 

```
% curl http://localhost:8000/core/example-tasks

{
  "documentLinks": [
    "/core/example-tasks/task-1",
    "/core/example-tasks/task-2",
    "/core/example-tasks/task-3"
  ],
  "documentCount": 3,
  "queryTimeMicros": 1997,
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "hostAtPort8000"
}
```

##8. Restart node

Now let's restart the node 8002 and set the membershipQuorum = 2:

```
% java -Dxenon.NodeState.membershipQuorum=2 -cp xenon-host/target/xenon-host-0.7.6-SNAPSHOT-jar-with-dependencies.jar com.vmware.xenon.services.common.ExampleServiceHost --sandbox=/tmp/xenon --port=8002 --peerNodes=http://127.0.0.1:8000,http://127.0.0.1:8001,http://127.0.0.1:8002 --id=hostAtPort8002
```

You'll see the node 8002's status is AVAILABLE.

```
% curl http://localhost:8000/core/node-groups/default

{
  ...
  "nodes": {
    "hostAtPort8000": {
      "status": "AVAILABLE",
      "membershipQuorum": 2,
      ...
    },
    "hostAtPort8001": {
      "status": "AVAILABLE",
      "id": "hostAtPort8001",
      "membershipQuorum": 2,
      ...
    },
    "hostAtPort8002": {
      "status": "AVAILABLE",
      "id": "hostAtPort8002",
      "membershipQuorum": 2,
      ...
    }
  },
  ...
}
```

##9. Verify synchronization

Send a GET to the example task factory of the restarted node, that all the tasks are there:

```
% curl http://localhost:8002/core/example-tasks?expand

{
  "documentLinks": [
    "/core/example-tasks/task-1",
    "/core/example-tasks/task-2",
    "/core/example-tasks/task-3"
  ],
  "documents": {
    "/core/example-tasks/task-3": {
      ...
      "taskInfo": {
        "stage": "FINISHED",
        "isDirect": false
      },
      ...
      "documentOwner": "hostAtPort8000"
    },
    "/core/example-tasks/task-1": {
      ...
      "taskInfo": {
        "stage": "FINISHED",
        "isDirect": false
      },
      ...
      "documentOwner": "hostAtPort8001"
    },
    "/core/example-tasks/task-2": {
      ...
      "taskInfo": {
        "stage": "FINISHED",
        "isDirect": false
      },
      ...
      "documentOwner": "hostAtPort8000"
    }
  },
  ...
}
```

Check each task service's status, they are all finished.

#Summary

In this tutorial, we have introduced the following key concepts:

* Service host arguments


* Synchronization behaviors


* Membership quorum