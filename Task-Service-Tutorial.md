# 1.0 Overview

This page gives you an introduction to writing a task service in Xenon. A task service will perform long-running tasks on behalf of a client (a user or another Xenon service). Because Xenon is architected to be highly scalable and asynchronous, a service should not delay its response while a long-running task is running. Instead, it should accept the task and allow clients to query for the results later. 

# 1.1 Task Service Workflow

The workflow for a task service is simple, but may be surprising if you have not seen the pattern before. 

1. A client (user or another Xenon service) does a POST to the task factory service to create a task. The POST will include all parameters needed to describe the task. 
2. The task factory service creates the task service
3. The task factory service will go through a series of steps. For each step, it will:
  1. Make some action. In Xenon, this will generally be an asynchronous action that completes at some future time
  2. When the action completes, update the state of the task service by doing a PATCH back to the service. 
  3. When the PATCH is processed, this process repeats with the next action

__IMAGE HERE SOON__

Task services are good examples of the uniform use of REST throughout Xenon: all state changes and queries between services happen via REST. A service does not treat a request differently if it comes from an external client, another service, or itself. 

In a strict technical sense, requests are often not "REST" in that when services are running in the same process, they are optimized, in-process communication instead of HTTP. However, that distinction is transparent to the author of a service: requests are handled identically. 

# 1.2 Assumptions

This tutorial assumes that you have gone through the introductory [Example Service Tutorial](Example-Service-Tutorial) as well as the [Introduction to Service Queries](./Introduction-to-Service-Queries). 

This tutorial assumes you have installed Xenon and can run it from the command-line. It also assumes you have installed the "curl" command-line tool. If you don't want to use curl, the examples should be easily converted to use another tool, such as Postman. It also assumes you have installed the [jq tool](https://stedolan.github.io/jq/), which we use for formatting JSON responses from Xenon. If you do not have it installed, you can simply remove it from the commands below. 

# 2.0 Starting the Xenon host

We are going to start the Xenon host differently than the previous tutorials:

1. We are going to start a different host, the ExampleServiceHost. This host provides extra arguments to make it easier to work with authorization.
2. We are going to provide an argument for the sandbox (where all service documents are stored), instead of relying on the Java temporary directory. When the host is not running, you can delete the sandbox if you need to get back to "factory settings". 

Please note that you will need to change the name of the JAR file to match the version of Xenon you have. 

```
% java -cp xenon-host/target/xenon-host-0.4.1-SNAPSHOT-jar-with-dependencies.jar com.vmware.xenon.services.common.ExampleServiceHost --sandbox=/tmp/xenon
[0][I][1452210831944][ExampleServiceHost:8000][startImpl][ServiceHost/1161b183 listening on 127.0.0.1:8000]
```

# 3.0 The ExampleTaskService

The ExampleTaskService is a service that will delete all example service documents. This is a two step process:

1. Make a query (POST to the QueryTaskService) for all example service documents
2. Delete all of them. 

This will usually run rather quickly (unless you have a huge number of example services to delete), but is a good illustration of how an ExampleTask is written. 

Here is an illustration of what the ExampleTaskService does:

__IMAGE HERE SOON__

# 4.0 Using the ExampleTaskService

Before we look at the code, let's see what it's like to use the ExampleTaskService. This assumes you have started up the ExampleHost, as described above.

## 4.1 Create example services
In order to delete example services, we first need to have some example services. Let's do that:

_File:_ example-1.body
```
{
  "name": "example-1",
  "counter": 1
}
```

_File:_ example-2.body
```
{
  "name": "example-1",
  "counter": 1
}
```

Create the examples:
```
% curl -s -X POST -d@example-1.body http://localhost:8000/core/examples -H "Content-Type: application/json" | jq .
{
  "keyValues": {},
  "counter": 1,
  "name": "example-1",
  "documentVersion": 0,
  "documentEpoch": 0,
  "documentKind": "com:vmware:xenon:services:common:ExampleService:ExampleServiceState",
  "documentSelfLink": "/core/examples/f67fc854-8efa-4672-b62d-e1eb4f4a30d8",
  "documentUpdateTimeMicros": 1452213596649000,
  "documentUpdateAction": "POST",
  "documentExpirationTimeMicros": 0,
  "documentOwner": "e7d748e2-597e-466e-a5ec-ec21b6838e7c",
  "documentAuthPrincipalLink": "/core/authz/guest-user",
  "documentTransactionId": ""
}

% curl -s -X POST -d@example-2.body http://localhost:8000/core/examples -H "Content-Type: application/json" | jq .
{
  "keyValues": {},
  "counter": 2,
  "name": "example-2",
  "documentVersion": 0,
  "documentEpoch": 0,
  "documentKind": "com:vmware:xenon:services:common:ExampleService:ExampleServiceState",
  "documentSelfLink": "/core/examples/91fc7d06-2b8c-4106-ba7f-f0d258f0242c",
  "documentUpdateTimeMicros": 1452213602148002,
  "documentUpdateAction": "POST",
  "documentExpirationTimeMicros": 0,
  "documentOwner": "e7d748e2-597e-466e-a5ec-ec21b6838e7c",
  "documentAuthPrincipalLink": "/core/authz/guest-user",
  "documentTransactionId": ""
}
```

Verify that you can see the example services:
```
% curl http://localhost:8000/core/examples
{
  "documentLinks": [
    "/core/examples/f67fc854-8efa-4672-b62d-e1eb4f4a30d8",
    "/core/examples/91fc7d06-2b8c-4106-ba7f-f0d258f0242c"
  ],
  "documentCount": 2,
  "queryTimeMicros": 4999,
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "e7d748e2-597e-466e-a5ec-ec21b6838e7c"
}
```

## 4.2 Run a task

Running the task is easy by doing a POST to the factory service. The body of the POST will contain any needed parameters.

The ExampleTaskService does not require any parameters, but just deletes all example services that it is authorized to access. However, we do have one optional parameter, which is the time (in seconds) for the task to delete itself. Technically, this parameter is not required, because the client can just set the documentExpirationTimeMicros field for a time in the future, and the service will be deleted. Because that field is a pain to use in a tutorial (it's microseconds since January 1, 1970), we've added a parameter which is the number of seconds after which the task should delete itself. The task will set the documentExpirationTimeMicros for us. 

_File:_ task.post
```
{
  "taskLifetime": 5
}
```

Then create the task with a POST. Note that the response tells you that the task is started (taskInfo.stage) and querying for the examples (subStage):

```
% curl -s -X POST -d@task.body http://localhost:8000/core/example-tasks -H "Content-Type: application/json" | jq . 
{
  "taskLifetime": 5,
  "taskInfo": {
    "stage": "STARTED",
    "isDirect": false
  },
  "subStage": "QUERY_EXAMPLES",
  "documentVersion": 0,
  "documentKind": "com:vmware:xenon:services:common:ExampleTaskService:ExampleTaskServiceState",
  "documentSelfLink": "/core/example-tasks/7b0501a2-44ae-4b9e-8899-9a025a3b0416",
  "documentUpdateTimeMicros": 1452213703216002,
  "documentExpirationTimeMicros": 1452213708217002,
  "documentOwner": "e7d748e2-597e-466e-a5ec-ec21b6838e7c"
}
```

If we wait a second and query the task, we'll see that it's completed, and we'll also get to peek at the task's internal state, which includes the results of the query it did. The output has been trimmed of some of the standard fields for simplicity:

```
curl -s 'http://localhost:8000/core/example-tasks?expand' | jq . 
{
  "documentLinks": [
    "/core/example-tasks/7b0501a2-44ae-4b9e-8899-9a025a3b0416"
  ],
  "documents": {
    "/core/example-tasks/7b0501a2-44ae-4b9e-8899-9a025a3b0416": {
      "taskLifetime": 5,
      "taskInfo": {
        "stage": "FINISHED",
        "isDirect": false
      },
      "subStage": "DELETE_EXAMPLES",
      "exampleQueryTask": {
        "taskInfo": {
          "stage": "FINISHED",
          "isDirect": true,
          "durationMicros": 3000
        },
        "querySpec": {
          "query": {
            "occurance": "MUST_OCCUR",
            "term": {
              "propertyName": "documentKind",
              "matchValue": "com:vmware:xenon:services:common:ExampleService:ExampleServiceState",
              "matchType": "TERM"
            }
          },
          "resultLimit": 2147483647,
          "options": []
        },
        "results": {
          "documentLinks": [
            "/core/examples/f67fc854-8efa-4672-b62d-e1eb4f4a30d8",
            "/core/examples/91fc7d06-2b8c-4106-ba7f-f0d258f0242c"
          ],
          ... output trimmed ...
        },
        "indexLink": "/core/document-index",
        ... output trimmed ...
      },
      "documentVersion": 3,
      "documentKind": "com:vmware:xenon:services:common:ExampleTaskService:ExampleTaskServiceState",
      "documentSelfLink": "/core/example-tasks/7b0501a2-44ae-4b9e-8899-9a025a3b0416",
      "documentExpirationTimeMicros": 1452213708217002,
      ... output trimmed ...
    }
  },
  "documentCount": 1,
  ... output trimmed ...
}
```

We can also see that the example have been deleted:

```
% curl http://localhost:8000/core/examples
{
  "documentLinks": [],
  "documentCount": 0,
  "queryTimeMicros": 1,
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "e7d748e2-597e-466e-a5ec-ec21b6838e7c"
}
```

If we wait a few more seconds, the task will also be removed:

```
% curl -s 'http://localhost:8000/core/example-tasks?expand' | jq . 
{
  "documentLinks": [],
  "documents": {},
  "documentCount": 0,
  "queryTimeMicros": 999,
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "e7d748e2-597e-466e-a5ec-ec21b6838e7c"
}
```

# 5.0 Writing a Task Service in Java
