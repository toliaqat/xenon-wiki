# 1.0 Overview

This page gives you an introduction to writing a task service in Xenon. A task service will perform long-running tasks on behalf of a client (a user or another Xenon service). 

Because Xenon is architected to be highly scalable and asynchronous, a service should not delay its response while a long-running task is running. Instead, it should accept the task and allow clients to query or subscribe for the results. That is exactly what a task service does and what you can learn in this tutorial.

# 1.1 Task Service Workflow

The workflow for a task service is simple, but may be surprising if you have not seen the pattern before. 

1. A client (user or another Xenon service) does a POST to the task factory service to create a task. The POST will include all parameters needed to describe the task. 
2. The task factory service creates the task service
3. The task factory service will go through a series of steps. It models a finite state machine. For each step, it will:
  1. Make some action. In Xenon, this will generally be an asynchronous action that completes at some future time
  2. When the action completes, update the state of the task service by doing a PATCH back to itself. 
  3. When the PATCH is processed, this process repeats with the next action

The image below illustrates this pattern. Please note that in the diagram, the client only requests the state after it's completed, but it can request the state at any time. 

[[images/task-service/task-service.png]]

Task services are good examples of the uniform use of REST throughout Xenon: all state changes and queries between services happen via REST. A service does not treat a request differently if it comes from an external client, another service, or itself. For a task service, these requests include the POST that created the task as well as the self PATCH's that update the task as it progresses.

In a strict technical sense, requests are often not using HTTP because when services are running in the same process, they are optimized, in-process communication instead of going on the network/sockets. However, that distinction is transparent to the author of a service: requests are handled identically whether they arrive over an HTTP connection or from another in-process service.

## Direct Task work flow

A client can chose the "synchronous* REST pattern when creating a task:
 * client POSTs to task factory using POST and body that contains taskInfo.isDirect = true
 * Assuming task factory service was started using *TaskFactoryService.create(..)*, the default task factory
   creates a new POST, transfers the body and creates the TaskService instance
 * The task factory subscribes to the child task, which is operating in asynchronous mode
 * When the task service (child) is complete, the task factory completes the original client request

To the client, this was a simple POST, where the response is received when the task is done!

See [task factory service](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/TaskFactoryService.java#L95) for implementation details

## Starting a task factory

```
host.startFactory(() -> TaskFactoryService.create(ExampleTaskService.class),
                ExampleTaskService.FACTORY_LINK);
```
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

[[images/task-service/example-task-service.png]]

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
  "name": "example-2",
  "counter": 2
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

You start a task by doing a POST to the factory service. The body of the POST will contain any needed parameters.

The ExampleTaskService does not require any parameters, but just deletes all example services that it is authorized to access. However, we do have one optional parameter, which is the time (in seconds) for the task to delete itself. Technically, this parameter is not required, because the client can just set the documentExpirationTimeMicros field for a time in the future, and the task service will be deleted. Because that field is a pain to use in a tutorial (it's microseconds since January 1, 1970), we've added a parameter which is the number of seconds after which the task should delete itself. The task will set the documentExpirationTimeMicros for us. 

_File:_ task.body
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

If we wait a second and query the task, we will see that it has finished, and we also get to peek at the task's internal state, which includes the results of the query it did. The output has been trimmed of some of the standard fields for simplicity:

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

If we wait a few more seconds, the task will also be removed because it has expired:

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

The full code for the task factory and service can be found at:
* [ExampleTaskService.java](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/ExampleTaskService.java)
* [TaskService.java](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/TaskService.java) - the superclass that most task service implementations (including `ExampleTaskService`) should extend from

## 5.1 The task factory service

Similar to the [ExampleService](Example-Service-Tutorial), the task factory service is created through a static helper within ExampleTaskService.java:

```java
public static final String FACTORY_LINK = ServiceUriPaths.CORE + "/example-tasks";

/**
 * Create a default factory service that starts instances of this task service on POST.
 */
public static FactoryService createFactory() {
    return FactoryService.create(ExampleTaskService.class, ExampleTaskServiceState.class,
            ServiceOption.IDEMPOTENT_POST, ServiceOption.INSTRUMENTATION);
}
```

The task factory service is simple and identical in form to most other factory services because the functionality is contained within the base FactoryService class. The two essential pieces of this code are the definition of the FACTORY_LINK URI (we use /core/example-tasks) and the ServiceOptions. Note that the URI must be named FACTORY_LINK: it is used by the buildFactoryUri() method by Java clients that wish to make a POST to the factory.

## 5.2 The task service
The task service is where all of the interesting code is. We do not have the full code here; see [ExampleTaskService.java](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/ExampleTaskService.java) for details. We will highlight the most interesting parts. 

### 5.2.1 The state of the task
We must define the state of the task. Here we have one parameter for users to set and several for the task to keep track of what it is doing. We also extend from `TaskService.TaskServiceState` to inherit common task details, such as a `TaskState` and a failure message.

A few things to note:
* We use the standard [TaskState](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/common/TaskState.java), which defines the overall progress through the task. While you are not required to use TaskState, we strongly encourage it to provide commonality between all tasks services. The TaskState only has a few stages: CREATED, STARTED, FINISHED, FAILED, CANCELLED. Most tasks will spend their working time in the STARTED state and will indicate their progress through a SubStage.
* The UsageOption AUTO_MERGE_IF_NOT_NULL makes it easier to handle PATCH requests because the state changes can be merged for you by merging the changes automatically when you call `Utils.mergeWithState()`

```java
public static class ExampleTaskServiceState extends TaskService.TaskServiceState {

    /**
     * Time in seconds before the task expires
     *
     * Technically, this isn't needed: clients can just set documentExpirationTimeMicros.
     * However, this makes tutorials easier: command-line clients can just set this to
     * a constant instead of calculating a future time
     */
    @UsageOption(option = PropertyUsageOption.AUTO_MERGE_IF_NOT_NULL)
    public Long taskLifetime;

    /**
     * The current substage. See {@link SubStage}
     */
    @UsageOption(option = PropertyUsageOption.AUTO_MERGE_IF_NOT_NULL)
    public SubStage subStage;

    /**
     * The query we make to the Query Task service, and the result we
     * get back from it.
     */
    @UsageOption(option = PropertyUsageOption.AUTO_MERGE_IF_NOT_NULL)
    public QueryTask exampleQueryTask;
}
```

Note that the same options can be set at runtime, if the service author overrides getDocumentTemplate(). See the ExampleService for how this is done.

Here is our definition of the sub stage. This task only has two sub stages:

```java
public enum SubStage {
    QUERY_EXAMPLES, DELETE_EXAMPLES
}
```

### 5.2.2 Creating the TASK

When the factory service receives the POST to make the task, the task service's method `handleStart()` will be called (which is implemented in the `TaskService` base class). It is passed an Operation; this is the POST operation, and we can examine the body of the operation to see what to do. 

Here are a couple of important points that the `TaskService` base class takes care of for you:

1. As soon as it does a _quick_ validation, we send our response to the POST by calling _complete()_ on the operation. After that, we will initialize our state and PATCH ourselves. Note that if the client _immediately_ does a GET on the service, they may not see the initialized state yet. This isn't likely in practice for clients, but it may happen for in-process clients. 
2. Once the state is initialized, we immediately do a self PATCH. That PATCH will trigger the first step of work, and future PATCH's will continue the work. We do the self PATCH before doing any work to ensure that our state is updated.
3. `TaskService` provides reasonable defaults for validation and initialization of the base `TaskServiceState`-related data (all of which can be easily overridden).
4. Note that special care should be taken for restartable tasks: A service host might stop, restart, which will initiate
restart of all persisted services. Task services should check the state passed in the POST in handleStart, and only self patch if the supplied state is in the proper stage. For example, if the task is already marked as **FINISHED**, there is no
need to self patch and kick of the state machine again.

`TaskService.java` , where `T` is the class of the actual task's state (such as `ExampleTaskService.ExampleTaskServiceState`)
```java
/**
 * This handles the initial {@code POST} that creates the task service. Most subclasses won't
 * need to override this method, although they likely want to override the {@link
 * #validateStartPost(Operation)} and {@link #initializeState(TaskServiceState, Operation)}
 * methods.
 */
@Override
public void handleStart(Operation taskOperation) {
    T task = validateStartPost(taskOperation);
    if (task == null) {
        return;
    }
    taskOperation.complete();

    if (!ServiceHost.isServiceCreate(taskOperation)
                || (task.taskInfo != null && !TaskState.isCreated(task.taskInfo))) {
        // Skip self patch to STARTED if this is a restart operation, or, task stage is
        // other than CREATED.
        // Tasks that handle restart should override handleStart and decide if they should
        // continue processing on restart, or fail
        return;
    }

    initializeState(task, taskOperation);
    sendSelfPatch(task);
}

/**
 * Ensure that the initial input task is valid. Subclasses might want to override this
 * implementation to also validate their {@code SubStage}.
 */
protected T validateStartPost(Operation taskOperation) {
    T task = getBody(taskOperation);

    if (!taskOperation.hasBody()) {
        taskOperation.fail(new IllegalArgumentException("POST body is required"));
        return null;
    }

    if (!ServiceHost.isServiceCreate(taskOperation)) {
        // we apply validation only on the original, client issued POST, not operations
        // caused by host restart
        return task;
    }

    if (task.taskInfo != null) {
        taskOperation.fail(new IllegalArgumentException(
                "Do not specify taskBody: internal use only"));
        return null;
    }

    // Subclasses might also want to ensure that their "SubStage" is not specified also
    return task;
}

/**
 * Initialize the task with default values. Subclasses might want to override this
 * implementation to initialize their {@code SubStage}
 */
protected void initializeState(T task, Operation taskOperation) {
    task.taskInfo = new TaskState();
    task.taskInfo.stage = TaskState.TaskStage.STARTED;

    // Put in some default expiration time if it hasn't been provided yet.
    if (task.documentExpirationTimeMicros == 0) {
        setExpirationFromMinutes(task, DEFAULT_EXPIRATION_MINUTES);
    }

    // Subclasses should initialize their "SubStage"...
    taskOperation.setBody(task);
}
```

As the javadoc mentions, most task implementations will want to override the `validateStartPost()` and `initializeState()` methods to provide our task-specific logic... as seen below:

`ExampleTaskService.java`
```java
@Override
protected ExampleTaskServiceState validateStartPost(Operation taskOperation) {
    ExampleTaskServiceState task = super.validateStartPost(taskOperation);

    if (ServiceHost.isServiceCreate(taskOperation)) {
        // apply validation only for the initial creation POST, not restart. Alternatively,
        // this code can exist in the handleCreate method
        if (task.subStage != null) {
            taskOperation.fail(
                    new IllegalArgumentException("Do not specify subStage: internal use only"));
            return null;
        }
        if (task.exampleQueryTask != null) {
            taskOperation.fail(
                new IllegalArgumentException("Do not specify taskBody: internal use only"));
            return null;
        }
    }
    if (task.taskLifetime != null && task.taskLifetime <= 0) {
        taskOperation.fail(
                new IllegalArgumentException("taskLifetime must be positive"));
        return null;
    }

    return task;
}

@Override
protected void initializeState(ExampleTaskServiceState task, Operation taskOperation) {
    super.initializeState(task, taskOperation);
    task.subStage = SubStage.QUERY_EXAMPLES;

    if (task.taskLifetime != null) {
        task.documentExpirationTimeMicros = Utils.getNowMicrosUtc()
                + TimeUnit.SECONDS.toMicros(task.taskLifetime);
    } else if (task.documentExpirationTimeMicros != 0) {
        task.documentExpirationTimeMicros = Utils.getNowMicrosUtc()
                + TimeUnit.SECONDS.toMicros(DEFAULT_TASK_LIFETIME);
    }
}
```

### 5.2.2 Doing the work
All of the work of the task service is done in response to PATCH's. When our task service receives a PATCH, the handlePatch() method is called. Ours looks like the code below. 

1. Note that we respond to the PATCH as soon as we ensure it is valid. Just like the creation of the task, we respond immediately before doing the work. 
2. Note that all of our work is in the STARTED stage

```java
@Override
public void handlePatch(Operation patch) {
    ExampleTaskServiceState currentTask = getState(patch);
    ExampleTaskServiceState patchBody = getBody(patch);
    if (!validateTransition(patch, currentTask, patchBody)) {
        return;
    }

    updateState(currentTask, patchBody);
    patch.complete();

    switch (patchBody.taskInfo.stage) {
    case CREATED:
        // Won't happen: validateTransition reports error
        break;
    case STARTED:
        handleSubstage(patchBody);
        break;
    case CANCELLED:
        logInfo("Task canceled: not implemented, ignoring");
        break;
    case FINISHED:
        logInfo("Task finished successfully");
        break;
    case FAILED:
        logWarning("Task failed: %s", (patchBody.failureMessage == null ? "No reason given"
                : patchBody.failureMessage));
        break;
    default:
        logWarning("Unexpected stage: %s", patchBody.taskInfo.stage);
        break;
    }
}

private void handleSubstage(ExampleTaskServiceState task) {
    switch (task.subStage) {
    case QUERY_EXAMPLES:
        handleQueryExamples(task);
        break;
    case DELETE_EXAMPLES:
        handleDeleteExamples(task);
        break;
    default:
        logWarning("Unexpected sub stage: %s", task.subStage);
        break;
    }
}
```

NOTE: The `updateState()` method implementation is already in the base `TaskService` class:
```java
    /**
     * This updates the state of the task. Note that we are merging information from the
     * PATCH into the current task. Because we are merging into the current task (it's the
     * same object), we do not need to explicitly save the state: that will happen when
     * we call patch.complete()
     */
    private void updateState(ExampleTaskServiceState currentTask, ExampleTaskServiceState patchBody) {
        Utils.mergeWithState(getDocumentTemplate().documentDescription, currentTask, patchBody);

        // Take the new document expiration time
        if (currentTask.documentExpirationTimeMicros == 0) {
            currentTask.documentExpirationTimeMicros = patchBody.documentExpirationTimeMicros;
        }
    }
```

Our work is in two separate methods. Let's briefly look at them. 

To find all of the query examples, we send a POST to the query task work. This is a task worker, just like this task worker. It's a little unusual though, because queries are so frequent and usually very quick: we can request an immediate response instead of waiting for it to complete. This is called a direct task. 

Note that this method starts asynchronous work. If it needs to do long-running, CPU-intensive work, it's best to run it in another thread. When the work completes, it sends a PATCH back to the task worker to proceed with the next step. This PATCH updates the tasks state with the results of the query, so the next step can use the results. 

```java
private void handleQueryExamples(ExampleTaskServiceState task) {
    // Create a query for "all documents with kind ==
    // com:vmware:xenon:services:common:ExampleService:ExampleServiceState"
    Query exampleDocumentQuery = Query.Builder.create()
	    .setTerm(ServiceDocument.FIELD_NAME_KIND,
		    Utils.buildKind(ExampleServiceState.class))
	    .build();
    task.exampleQueryTask = QueryTask.Builder.createDirectTask()
	    .setQuery(exampleDocumentQuery)
	    .build();

    // Send the query to the query task service.
    // When we get a response, advance to the next substage, deleting examples
    // Note that we inherited the authorization context of the incoming patch, so
    // we will only see documents that can be seen by the requesting user.
    // The same is true of our completion: we'll continue to use the same authorization
    // context
    URI queryTaskUri = UriUtils.buildUri(this.getHost(), ServiceUriPaths.CORE_QUERY_TASKS);
    Operation queryRequest = Operation.createPost(queryTaskUri)
	    .setBody(task.exampleQueryTask)
	    .setCompletion(
		    (op, ex) -> {
                        if (ex != null) {
			    logWarning("Query failed, task will not finish: %s",
				    ex.getMessage());
			    return;
                        }
                        // We extract the result of the task because DELETE_EXAMPLES will use
                        // the list of documents found
                        task.exampleQueryTask = op.getBody(QueryTask.class);
                        sendSelfPatch(task, TaskStage.STARTED, SubStage.DELETE_EXAMPLES);
		    });
    sendRequest(queryRequest);
}
```

The step to delete the example services is similar in concept, but it uses something you may not have seen before: [OperationJoin](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/common/OperationJoin.java). This allows you to send multiple operations in parallel (batching can be used optionally), and receive just one completion when all of them finish. This greatly simplifies our deletion of the example services:

```java
private void handleDeleteExamples(ExampleTaskServiceState task) {
    if (task.exampleQueryTask.results == null) {
        sendSelfFailurePatch(task, "Query task service returned null results");
        return;
    }

    if (task.exampleQueryTask.results.documentLinks == null) {
        sendSelfFailurePatch(task, "Query task service returned null documentLinks");
        return;
    }
    if (task.exampleQueryTask.results.documentLinks.size() == 0) {
        logInfo("No example service documents found, nothing to do");
        sendSelfPatch(task, TaskStage.FINISHED, null);
    }

    List<Operation> deleteOperations = new ArrayList<>();
    for (String exampleService : task.exampleQueryTask.results.documentLinks) {
        URI exampleServiceUri = UriUtils.buildUri(this.getHost(), exampleService);
        Operation deleteOp = Operation.createDelete(exampleServiceUri);
        deleteOperations.add(deleteOp);
    }

    // OperationJoin lets us do a set of operations in parallel. If we wanted to,
    // we could specify a batch size to limit the parallelism. We'll receive one
    // completion when all the operations complete.
    OperationJoin operationJoin = OperationJoin.create();
    operationJoin
	    .setOperations(deleteOperations)
	    .setCompletion((ops, exs) -> {
                if (exs != null && !exs.isEmpty()) {
		    sendSelfFailurePatch(task, String.format("%d deletes failed", exs.size()));
                    return;
                } else {
                    sendSelfPatch(task, TaskStage.FINISHED, null);
                }
            }).sendWith(this);
}
```

One last important code to understand is the self PATCH. Fortunately, it's straightforward. Note that we need to know the tasks URI, but this is provided for you with a method called `getUri()`. 

```java
/**
 * Send ourselves a PATCH that will indicate failure
 */
private void sendSelfFailurePatch(ExampleTaskServiceState task, String failureMessage) {
    task.failureMessage = failureMessage;
    sendSelfPatch(task, TaskStage.FAILED, null);
}

/**
 * Send ourselves a PATCH that will advance to another step in the task workflow to the
 * specified stage and substage.
 */
private void sendSelfPatch(ExampleTaskServiceState task, TaskStage stage, SubStage subStage) {
    if (task.taskInfo == null) {
        task.taskInfo = new TaskState();
    }
    task.taskInfo.stage = stage;
    task.subStage = subStage;
    sendSelfPatch(task);
}

/**
 * Send ourselves a PATCH. The caller is responsible for creating the PATCH body
 */
private void sendSelfPatch(ExampleTaskServiceState task) {
    Operation patch = Operation.createPatch(getUri())
	    .setBody(task)
	    .setCompletion(
                    (op, ex) -> {
                        if (ex != null) {
			    logWarning("Failed to send patch, task has failed: %s",
				    ex.getMessage());
                        }
		    });
    sendRequest(patch);
}
```

# 6.0 Subscriptions
You may have noticed that our command-line example used polling to find out when a task finished. Within Xenon, you can write code that instead subscribes to a task service and will receive a notification every time it changes its state. This is more efficient than polling. 

There is an example of this in [TestExampleTaskService.java](https://github.com/vmware/xenon/blob/master/xenon-common/src/test/java/com/vmware/xenon/services/common/TestExampleTaskService.java). The essential part of the code has two parts. 

First, you need a completion handler that will be notified of every change. The completion handler takes one parameter, which is the operation that is sent to you. The body of the operation will contain the change that was made, such as PATCH or DELETE. 

```java
/**
 * This creates a lambda to receive notifications. It's meant as a demonstration of how
 * to receive notifications. 
 */
private Consumer<Operation> createNotificationTarget() {

    Consumer<Operation> notificationTarget = (update) -> {
        update.complete();

        if (!update.hasBody()) {
	    // This is probably a DELETE
	    this.host.log(Level.INFO, "Got notification: %s", update.getAction());
	    return;
        }

        ExampleTaskServiceState taskState = getBody(update);
        this.host.log(Level.INFO, "Got notification: %s", taskState);
        String stage = "Unknown";
        String substage = "Unknown";
        if (taskState.taskInfo != null && taskState.taskInfo.stage != null) {
	    stage = taskState.taskInfo.stage.toString();
        }
        if (taskState.subStage != null) {
	    substage = taskState.subStage.toString();
        }
        this.host.log(Level.INFO,
                "Received task notification: %s, stage = %s, substage = %s",
                update.getAction(), stage, substage);
    };
    return notificationTarget;
}
```

Next, you need to request that notifications are sent. Here's what it looks like. Not that this has been modified slightly to remove test-specific code. 

```java
/**
 * Subscribe to notifications from the task.
 *
 * Note that in this short-running test, we are not guaranteed to get all notifications:
 * we may subscribe after the task has completed some or all of its steps. However, we
 * usually get all notifications.
 *
 */
private void subscribeTask(String taskPath, Consumer<Operation> notificationTarget)
        throws Throwable {
    URI taskUri = UriUtils.buildUri(this.host, taskPath);
    Operation subscribe = Operation.createPost(taskUri)
	    //.setCompletion(...) will be called when your subscription succeeds or fails
	    .setReferer(this.host.getReferer());

    // the notificationTarget is the completion we made above
    this.host.startSubscriptionService(subscribe, notificationTarget);
}
```

*NOTE*: If you are subscribing to a task that might finish _extremely_ fast, there's a special "replay-state" flag you should set. Without this special flag, you might subscribe to a task that is already FINISHED (which means your notification target will never be invoked). If you setup your subscription with this "replay-state" flag, you can be guaranteed to get at least one notification with the latest state. The code is slightly different than above:

```java
    this.host.startSubscriptionService(subscribe, notificationTarget, ServiceSubscriptionState.ServiceSubscriber.create(true));
```

For more information on subscriptions, see the description on the [Programming Model](Programming-Model#subscriptions) page. 

# 7.0 Writing Unit Tests for a Task Service in Java

Unit tests are critical components in software development, and Xenon provides ways to easily implement unit tests to validate your task-specific logic. There are two testing classes you can extend when writing your unit tests:

- [BasicTestCase](https://github.com/vmware/xenon/blob/master/xenon-common/src/test/java/com/vmware/xenon/common/BasicTestCase.java) - This base class supports starting up a VerificationHost before each test is run, and then tearing it down after the test is completed
- [BasicReusableHostTestCase](https://github.com/vmware/xenon/blob/master/xenon-common/src/test/java/com/vmware/xenon/common/BasicReusableHostTestCase.java) - This base class is similar, but only starts the VerificationHost once in its `@BeforeClass` method (and tears it down after all tests have run via its `@AfterClass` method)

Both methods are viable options; the reusable version will help speed up the running of your unit tests, but you'll need to take extra care that your unit tests are not modifying persisted states that other unit tests rely on.

## The VerificationHost

A crucial component to unit testing Xenon services is the [VerificationHost](https://github.com/vmware/xenon/blob/master/xenon-common/src/test/java/com/vmware/xenon/common/test/VerificationHost.java). This is a special `ServiceHost` for unit testing that allows developers to create synchronous test contexts around what is usually a completely asynchronous framework.

A `VerificationHost` has plenty of useful methods for unit tests to leverage, including:
- `testStart(long count)` - starts a "test context" with an expected `count` of iterations. This `count` is the number of iterations the test context expects `completeIteration()` to be called
- `testWait()` - blocks until all iterations of the "test context" have either completed or failed. `testWait(int)` also takes an integer for timeout seconds.
- When a "test context" has been started... there are three possible outcomes of each expected iteration:
  - `completeIteration()` - this means that an iteration of the test context has completed as expected by the creator of the unit test
  - `failIteration()` - the opposite; an iteration failed, which should be considered a test failure
  - if neither of the two are called within a specified timeout, then the test context times out (also considered a failure)

There are a ton of other helpful testing methods, which you can examine by looking at the source code.

## TestExampleTaskService

A great way to get a feel for writing unit tests for your task service implementation is to look at the existing [TestExampleTaskService](https://github.com/vmware/xenon/blob/master/xenon-common/src/test/java/com/vmware/xenon/services/common/TestExampleTaskService.java).

### The "happy" path test

Let's write a test to ensure that the `ExampleTaskService` properly deletes all Example ServiceDocuments. To implement this test, we need to:

1. Populate our document index with some ExampleService instances
2. Send a POST to create an `ExampleTaskService` task
3. Wait for the task to finish
4. Verify no Examples are left

> NOTE: The actual test in our repo does some more validation, but for the purposes of this wiki we will only focus on implementing this.

Here's how you could write such a test.

```java
    @Test
    public void testExampleTestServices() throws Throwable {
        // 1. Populate document index with some ExampleService instances
        createExampleServices();

        // 2. Send a POST to create our task. Verify task contains a URI (was created successfully)
        String[] taskUri = new String[1];
        CompletionHandler successCompletion = getCompletionWithUri(taskUri);
        ExampleTaskServiceState initialState = new ExampleTaskServiceState();
        sendFactoryPost(ExampleTaskService.class, initialState, successCompletion);
        assertNotNull(taskUri[0]);

        // 3. Wait for task to finish (ensure it ended up as FINISHED)
        ExampleTaskServiceState state = waitForFinishedTask(initialState.getClass(), taskUri[0]);
        assertEquals(TaskState.TaskStage.FINISHED, state.taskInfo.stage);

        // 4. Verify no examples are left
        validateNoServices();
    }

    /** Create a set of example services, so we can test that the ExampleTaskService cleans them up */
    private void createExampleServices() throws Throwable {
        URI exampleFactoryUri = UriUtils.buildFactoryUri(this.host, ExampleService.class);

        this.host.testStart(this.numServices);
        for (int i = 0; i < this.numServices; i++) {
            ExampleServiceState example = new ExampleServiceState();
            example.name = String.format("example-%s", i);
            Operation createPost = Operation.createPost(exampleFactoryUri)
                    .setBody(example)
                    .setCompletion(this.host.getCompletion());
            this.host.send(createPost);
        }
        this.host.testWait();
    }

    /** Verify that the task correctly cleaned up all the example services: none should be left. */
    private void validateNoServices() throws Throwable {
        URI exampleFactoryUri = UriUtils.buildFactoryUri(this.host, ExampleService.class);

        ServiceDocumentQueryResult exampleServices = this.host.getServiceState(null,
                ServiceDocumentQueryResult.class,
                exampleFactoryUri);

        assertNotNull(exampleServices);
        assertNotNull(exampleServices.documentLinks);
        assertEquals(exampleServices.documentLinks.size(), 0);
    }
```

You can see from above that there are a lot of functionality common to Xenon Task Services that you get for free so you can focus on your task-specific testing logic, including:
- `getCompletionWithUri` - provides a CompletionHandler that expects a successful response, and also sets the value of the URI (documentSelfLink) of the service instance
- `sendFactoryPost` - creates a service instance by sending a POST to a Service factory, providing the `state` to use for the body of the POST and the completion handler to use for processing the POST
- `waitForFinishedTask` - even though this task executes quickly, there is still some waiting we need to do before we can validate the final state. Other related methods include `waitForFailedTask` and `waitForTask`.
- `getServiceState` - returns the state of a requested Service from the document index

For more details about these helper test methods, see javadocs included in [VerificationHost](https://github.com/vmware/xenon/blob/master/xenon-common/src/test/java/com/vmware/xenon/common/test/VerificationHost.java).

### The "unhappy" path test

Before a task is successfully created, it needs to be validated. This is logic in the `validateStartPost()` method, and we likely want to write unit tests to ensure it's working properly. For our `ExampleTaskService`, we ensure the client cannot provide values like:
- a non-null `TaskStage` - such as `CREATED`. This should be set by the task logic itself, not by the body of the POST specified by the client
- a non-null `SubStage` - same reasons as above
- a negative value for `taskLifetime` 

Here's how you might want to test that this error handling is working as expected

```java
    @Test
    public void handleStartError_taskBody() throws Throwable {
        ExampleTaskServiceState badState = new ExampleTaskServiceState();
        badState.taskInfo = new TaskState();
        badState.taskInfo.stage = TaskState.TaskStage.CREATED;
        testExpectedHandleStartError(badState, IllegalArgumentException.class, "Do not specify taskBody: internal use only");
    }

    @Test
    public void handleStartErrors_subStage() throws Throwable {
        ExampleTaskServiceState badState = new ExampleTaskServiceState();
        badState.subStage = ExampleTaskService.SubStage.QUERY_EXAMPLES;
        testExpectedHandleStartError(badState, IllegalArgumentException.class, "Do not specify subStage: internal use only");
    }

    @Test
    public void handleStartErrors_exampleQueryTask() throws Throwable {
        ExampleTaskServiceState badState = new ExampleTaskServiceState();
        badState.exampleQueryTask = QueryTask.create(null);
        testExpectedHandleStartError(badState, IllegalArgumentException.class, "Do not specify exampleQueryTask: internal use only");
    }

    @Test
    public void handleStartErrors_taskLifetimeNegative() throws Throwable {
        ExampleTaskServiceState badState = new ExampleTaskServiceState();
        badState.taskLifetime = -1L;
        testExpectedHandleStartError(badState, IllegalArgumentException.class, "taskLifetime must be positive");
    }

    private void testExpectedHandleStartError(ExampleTaskServiceState badState,
            Class<? extends Throwable> expectedException, String expectedMessage) throws Throwable {
        Throwable[] thrown = new Throwable[1];
        Operation.CompletionHandler errorHandler = getExpectedFailureCompletionReturningThrowable(thrown);
        sendFactoryPost(ExampleTaskService.class, badState, errorHandler);

        assertNotNull(thrown[0]);

        String message = String.format("Thrown exception [thrown=%s] is not 'instanceof' [expected=%s]", thrown[0].getClass(), expectedException);
        assertTrue(message, expectedException.isAssignableFrom(thrown[0].getClass()));
        assertEquals(expectedMessage, thrown[0].getMessage());
    }
```

Again, notice how we can leverage a lot of base behavior for free, including:
- `getExpectedFailureCompletionReturningThrowable` -  provides a CompletionHandler that expects an exception in the response (and does a `failIteration` if the response did not result in an exception). It also sets the value of the exception so you can assert the exception (and message) were what you expected
- `sendFactoryPost` - notice how we can reuse this method for both "happy" and "unhappy" paths. The important thing to note is we are changing the completion handler we are passing.

### Pitfalls to Avoid

Unit testing distributed and asynchronous Xenon services requires careful understanding of what you can **deterministically** test, and testing scenarios which are **nondeterministic**. This is an important distinction and can lead to intermittent test failures which result from poorly implemented tests. While a "test context" does provide synchronous testing abilities, there are some things it cannot do.

Let's say you wanted to write a unit test to ensure that the `ExampleTaskService.DEFAULT_TASK_LIFETIME` was correctly used when a client didn't specify `taskLifetime`. You might write something like this when you are writing unit test code to create a new task (and then verify the expiration was defaulted correctly):

```java
    private String buggyCreateTaskAndValidateDefaultExpiration() throws Throwable {
        URI exampleTaskFactoryUri = UriUtils.buildFactoryUri(this.host, ExampleTaskService.class);

        String[] taskUri = new String[1];
        long[] initialExpiration = new long[1];
        ExampleTaskServiceState task = new ExampleTaskServiceState();
        Operation createPost = Operation.createPost(exampleTaskFactoryUri)
                .setBody(task)
                .setCompletion(
                        (op, ex) -> {
                            if (ex != null) {
                                this.host.failIteration(ex);
                                return;
                            }
                            ExampleTaskServiceState taskResponse = op.getBody(ExampleTaskServiceState.class);
                            taskUri[0] = taskResponse.documentSelfLink;
                            initialExpiration[0] = taskResponse.documentExpirationTimeMicros;
                            this.host.completeIteration();
                        });

        this.host.testStart(1);
        this.host.send(createPost);
        this.host.testWait();

        assertNotNull(taskUri[0]);

        // Since our task body didn't set expiration, the default from ExampleTaskService should be used
        long expectedExpiration = Utils.getNowMicrosUtc() + TimeUnit.SECONDS.toMicros(ExampleTaskService.DEFAULT_TASK_LIFETIME);
        long wiggleRoom = TimeUnit.SECONDS.toMicros(10); // ensure it's accurate within 10 seconds
        long minExpectedTime = expectedExpiration - wiggleRoom;
        long maxExpectedTime = expectedExpiration + wiggleRoom;
        long actual = initialExpiration[0];

        String msg = String.format(
                "Task's expiration is incorrect. [minExpected=%tc] [maxExpected=%tc] : [actual=%tc]",
                minExpectedTime, maxExpectedTime, actual);
        assertTrue(msg, actual >= minExpectedTime && actual <= maxExpectedTime);
        return taskUri[0];
    }
```

This looks innocent enough: The `createPost`'s completion handler saves what the initial expiration was... and then later asserts that it was set to the correct default. The unit test will likely pass when you run it also! However, this is a great example of a **nondeterministic** test that has a bug in it. This unit test might fail in some environments, which is why it should be avoided when writing tests.

The bug here hinges on the `Operation.complete()` method. `ExampleTaskService` inherits the base `TaskService.handleStart()` method, which looks like this:
```java
    public void handleStart(Operation taskOperation) {
        T task = validateStartPost(taskOperation);
        if (task == null) {
            return;
        }
        taskOperation.complete();

        initializeState(task, taskOperation); // NOTE: default expiration is set HERE
        sendSelfPatch(task);
    }
```

The POST body did not have any validation errors, so `validateStartPost()` completed successfully. Next, it then marks the operation as `complete()` before initializing its state. This is intentional; initializing state could take some time, and we don't need/want to block the client while state is being initialized since we know the request was valid.

However, once the operation is marked `complete()`, the client can get a response back including the current state of the task (which, in this case, really only has the `documentSelfLink` initialized since `initializeState` hasn't ran yet).

This test is nondeterministic because the test result will be different based on when the completion handler runs.

**Failure Scenario**: CompletionHandler runs immediately after `complete()`, so assert on expiration time will result in a failed test:
```java
    public void handleStart(Operation taskOperation) {
        T task = validateStartPost(taskOperation);
        if (task == null) {
            return;
        }
        taskOperation.complete();
        // FAILURE case: CompletionHandler is invoked here. documentExpirationTimeMicros is zero since `initializeState` hasn't defaulted it yet

        initializeState(task, taskOperation); // NOTE: default expiration is set HERE
        sendSelfPatch(task);
    }
```

**Success Scenario**: CompletionHandler runs after the `initializeState()` has run and the Service state has been PATCHed with the correct defaults... leading to a passing test
```java
    public void handleStart(Operation taskOperation) {
        T task = validateStartPost(taskOperation);
        if (task == null) {
            return;
        }
        taskOperation.complete();

        initializeState(task, taskOperation); // NOTE: default expiration is set HERE
        sendSelfPatch(task);
        // SUCCESS case: CompletionHandler is invoked here. documentExpirationTimeMicros has been defaulted because `initializeState` already ran
    }
```

Long story short: when writing unit tests for tasks (which often PATCH their state extremely quickly after they've been created), be sure the scenario you are testing is deterministic to avoid headaches of intermittently failing tests.

# 8.0 Further Reading

* [LuceneQueryTaskService](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/LuceneQueryTaskService.java) The implementation of /core/query-tasks. This is a significantly more complicated example of a task service
* [TestExampleTaskService.java](https://github.com/vmware/xenon/blob/master/xenon-common/src/test/java/com/vmware/xenon/services/common/TestExampleTaskService.java) Test code for validating the ExampleTestService.
* [Finite State Machine](./Finite-State-Machine): Some utilities for helping you maintain more complex state machines. 
