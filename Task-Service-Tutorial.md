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

You start a task by doing a POST to the factory service. The body of the POST will contain any needed parameters.

The ExampleTaskService does not require any parameters, but just deletes all example services that it is authorized to access. However, we do have one optional parameter, which is the time (in seconds) for the task to delete itself. Technically, this parameter is not required, because the client can just set the documentExpirationTimeMicros field for a time in the future, and the task service will be deleted. Because that field is a pain to use in a tutorial (it's microseconds since January 1, 1970), we've added a parameter which is the number of seconds after which the task should delete itself. The task will set the documentExpirationTimeMicros for us. 

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
* The factory service: [ExampleTaskFactoryService.java](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/ExampleTaskFactoryService.java)
* The task service: [ExampleTaskService.java](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/ExampleTaskService.java)

## 5.1 The task factory service
The task factory service is simple and identical in form to most other factory services. Most of the functionality is in the base FactoryService class so the derived class is concise. The two essential pieces of this code are the definition of the URI (we use /core/example-tasks) and the reference to the class that implements the service

```java
public class ExampleTaskFactoryService extends FactoryService {
    public static final String SELF_LINK = ServiceUriPaths.CORE + "/example-tasks";

    public ExampleTaskFactoryService() {
        super(ExampleTaskServiceState.class);
        super.toggleOption(ServiceOption.IDEMPOTENT_POST, true);
        super.toggleOption(ServiceOption.INSTRUMENTATION, true);
    }

    @Override
    public Service createServiceInstance() throws Throwable {
        return new ExampleTaskService();
    }
}
```

## 5.2 The task service
The task service is where all of the interesting code is. We do not have the full code here; see [ExampleTaskService.java](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/ExampleTaskService.java) for details. We will highlight the most interesting parts. 

### 5.2.1 The state of the task
We must define the state of the task. Here we have one parameter for users to set and several for the task to keep track of what it is doing. 

A few things to note:
* We use the standard [TaskState](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/common/TaskState.java), which defines the overall progress through the task. While you are not required to use TaskState, we strongly encourage it to provide commonality between all tasks services. The TaskState only has a few stages: CREATED, STARTED, FINISHED, FAILED, CANCELLED. Most tasks will spend their working time in the STARTED state and will indicate their progress through a SubStage.
* The UsageOption AUTO_MERGE_IF_NOT_NULL makes it easier to handle PATCH requests because the state changes can be merged for you by merging the changes automatically when you call `Utils.mergeWithState()`

```java
public static class ExampleTaskServiceState extends ServiceDocument {
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
     * This field shouldn't be manipulated by clients, but can be examined to see the progress
     * of the task
     */
    @UsageOption(option = PropertyUsageOption.AUTO_MERGE_IF_NOT_NULL)
    public TaskState taskInfo;
     /**
     * If taskInfo.stage == FAILED, this message will say why
     */
    @UsageOption(option = PropertyUsageOption.AUTO_MERGE_IF_NOT_NULL)
    public String failureMessage;

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
public static enum SubStage {
    QUERY_EXAMPLES, DELETE_EXAMPLES
}
```

### 5.2.2 Creating the TASK

When the factory service receives the POST to make the task, the task service's method handleStart will be called. It is passed an Operation; this is the POST operation, and we can examine the body of the operation to see what to do. 

A couple of important points:

1. As soon as we do a _quick_ validation, we send our response to the POST by calling _complete()_ on the operation. After that, we will initialize our state and PATCH ourselves. Note that if the client _immediately_ does a GET on the service, they may not see the initialized state yet. This isn't likely in practice for clients, but it may happen for in-process clients. 
2. Once the state is initialized, we immediately do a self PATCH. That PATCH will trigger the first step of work, and future PATCH's will continue the work. We do the self PATCH before doing any work to ensure that our state is updated. 

```java
/**
 * This handles the initial POST that creates the task service.
 */
@Override
public void handleStart(Operation taskOperation) {
    ExampleTaskServiceState task = validateStartPost(taskOperation);
    if (task == null) {
        return;
    }
    taskOperation.complete();
    initializeState(task, taskOperation);
    sendSelfPatch(task);
}
 /**
 * Ensure that the input task is valid.
 *
 * Technically we don't need to require a body since there are no parameters. However,
 * non-example tasks will normally have parameters, so this is an example of how they
 * could be validated.
 */
private ExampleTaskServiceState validateStartPost(Operation taskOperation) {
    if (!taskOperation.hasBody()) {
        taskOperation.fail(new IllegalArgumentException("POST body is required"));
        return null;
    }
     ExampleTaskServiceState task = taskOperation.getBody(ExampleTaskServiceState.class);
    if (task.taskInfo != null) {
        taskOperation.fail(
                new IllegalArgumentException("Do not specify taskBody: internal use only"));
        return null;
    }
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
    if (task.taskLifetime != null && task.taskLifetime <= 0) {
        taskOperation.fail(
                new IllegalArgumentException("taskLifetime must be positive"));
        return null;
    }
     return task;
}
 /**
 * Initialize the task
 *
 * We set it to be STARTED: we skip CREATED because we don't need the CREATED state
 * If your task does significant initialization, you may prefer to do it in the
 * CREATED state.
 */
private void initializeState(ExampleTaskServiceState task, Operation taskOperation) {
    task.taskInfo = new TaskState();
    task.taskInfo.stage = TaskState.TaskStage.STARTED;
    task.subStage = SubStage.QUERY_EXAMPLES;
     if (task.taskLifetime != null) {
        task.documentExpirationTimeMicros = Utils.getNowMicrosUtc()
                + TimeUnit.SECONDS.toMicros(task.taskLifetime);
    } else if (task.documentExpirationTimeMicros != 0) {
        task.documentExpirationTimeMicros = Utils.getNowMicrosUtc()
                + TimeUnit.SECONDS.toMicros(DEFAULT_TASK_LIFETIME);
    }
    taskOperation.setBody(task);
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
    ExampleTaskServiceState patchBody = patch.getBody(ExampleTaskServiceState.class);
     if (!validateTransition(patch, currentTask, patchBody)) {
        return;
    }

    updateState(patch, currentTask, patchBody);
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

    /**
     * This updates the state of the task. Note that we are merging information from the
     * PATCH into the current task. Because we are merging into the current task (it's the
     * same object), we do not need to explicitly save the state: that will happen when
     * we call patch.complete()
     */
    private void updateState(Operation patch,
            ExampleTaskServiceState currentTask,
            ExampleTaskServiceState patchBody) {
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

        ExampleTaskServiceState taskState = update.getBody(ExampleTaskServiceState.class);
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

For more information on subscriptions, see the description on the [Programming Model](Programming-Model#subscriptions) page. 

# 7.0 Further Reading

* [LuceneQueryTaskService](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/LuceneQueryTaskService.java) The implementation of /core/query-tasks. This is a significantly more complicated example of a task service
* [TestExampleTaskService.java](https://github.com/vmware/xenon/blob/master/xenon-common/src/test/java/com/vmware/xenon/services/common/TestExampleTaskService.java) Test code for validating the ExampleTestService.
* [Finite State Machine](./Finite-State-Machine): Some utilities for helping you maintain more complex state machines. 