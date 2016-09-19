# Overview

This tutorial demonstrates how to write a test using xenon testing framework.  

## Prerequisites

- [Example Service Tutorial](Example-Service-Tutorial)
- [Multi Node Tutorial](Multi-Node-Tutorial)
- [Testing Guide](Testing-Guide)


We use `ExampleService`, `ExampleTaskService`, and `ExampleServiceHost` to demonstrate the topic. These classes are also available in xenon testing framework module.

## Module dependency

All xenon testing framework classes are packaged in `xenon-common-[version]-tests.jar`.

**for maven:**

```xml
<dependency>
    <groupId>com.vmware.xenon</groupId>
    <artifactId>xenon-common</artifactId>
    <version>${xenon.version}</version>
    <classifier>tests</classifier>
    <scope>test</scope>
</dependency>
```

## Starting ServiceHost

To test `ExampleService` and `ExampleTaskService`, we use `ExampleServiceHost` which is a `ServiceHost` implementation that simply starts those two services.


```java
// use junit TemporaryFolder rule to create random working directory for ServiceHost
@Rule
public TemporaryFolder tempFolder = new TemporaryFolder();

private ExampleServiceHost createAndStartExampleServiceHost() throws Throwable {
    Arguments args = new Arguments();
    args.sandbox = tempFolder.newFolder().toPath();  // use random directory
    args.port = 0;  // use random port

    ExampleServiceHost host = new ExampleServiceHost();
    host.initialize(args);
    host.start();
    return host;
}
```

```java
ExampleServiceHost host1 = createAndStartExampleServiceHost();
ExampleServiceHost host2 = createAndStartExampleServiceHost();
```

## Waiting target service

Since `host1` and `host2` are independently started, they have to know each other by joining same node group.
Also, we have to wait the factory service to be ready in the node group because `ExampleService` and `ExampleTaskService` are `PERSISTENCE`, `OWNER_SELECTION` services.

```java
TestNodeGroupManager nodeGroup = new TestNodeGroupManager()
        .addHost(host1)
        .addHost(host2);

nodeGroup.joinNodeGroupAndWaitForConvergence();  // join to default node group
nodeGroup.waitForFactoryServiceAvailable("/core/examples");
```

## Sending request synchronously

Once node group is converged and factory service is ready for the replicated service, it is time to perform tests by creating service instances and validating them.  
In test, it certainly makes code readable if code is written in sequential manner; therefore, xenon test framework provides `TestRequestSender` that has methods to send `Operation(s)` and receive response(s) *synchronously*.


### Single request

First, let's make a POST request to the factory to create a service.
We use static `documentSelfLink` to make it easy to validate the target service.

```java
// create request sender using one of the hosts in the nodegroup
TestRequestSender sender = new TestRequestSender(nodeGroup.getHost());

ExampleServiceState postBody = new ExampleServiceState();
postBody.name = "foo";
postBody.counter = 0L;
postBody.documentSelfLink = "foo";  // static self link: /core/examples/foo

Operation post = Operation.createPost(host1, "/core/examples").setBody(postBody);

// send POST and wait for response, then validate
ExampleServiceState postResult = sender.sendAndWait(post, ExampleServiceState.class);
assertEquals("foo", postResult.name);
assertEquals("/core/examples/foo", postResult.documentSelfLink);
```


### Multiple request

`TestRequestSender` also provide methods to send multiple `Operations` asynchronously then wait until all of them returns response.

Let's send multiple PATCH requests and check the counter value.  
The implementation of `handlePut` in `ExampleService` makes sure the max of the value is set to the counter.

```java
List<Operation> patches = new ArrayList<>();
for (int i = 0; i < 20; i++) {
    ExampleServiceState patchBody = new ExampleServiceState();
    patchBody.name = "foo-" + i;
    patchBody.counter = (long) (i + 1);  // 1-20
    Operation patch = Operation.createPatch(host1, "/core/examples/foo").setBody(patchBody);
    patches.add(patch);
}
// send 20 PATCH requests in parallel and wait all responses to be successful
sender.sendAndWait(patches);

// verify latest state
Operation get = Operation.createGet(host1, "/core/examples/foo");
ExampleServiceState getResult = sender.sendAndWait(get, ExampleServiceState.class);
assertEquals(Long.valueOf(20), getResult.counter);
```


### Expect request failure

Sometimes, you want to expect the request to fail.  
`TestRequestSender#sendAndWaitFailure` expects failure response and returns `FailureResponse` object.

```java
// delete foo service
Operation delete = Operation.createDelete(host1, "/core/examples/foo").setBody("{}");
sender.sendAndWait(delete);

// verify call to foo returns 404
Operation getToDeleted = Operation.createGet(host1, "/core/examples/foo");
FailureResponse failureResponse = sender.sendAndWaitFailure(getToDeleted);
assertEquals(404, failureResponse.op.getStatusCode());
```

## Wait condition

In xenon, everything happens in async. Thus, you cannot simply expect work has done right after the operation has completed.  
While writing tests, it requires certain logic to check whether your target action has finished or not. `TestContext#waitFor` or `VerificationHost#waitFor` provide a mechanism to wait until supplied check logic satisfies the condition.

To demonstrate the wait condition, let's write another test case using `ExampleTaskService`.

First, let's prepare `host` and `sender`, then make a POST request to create a `ExampleTaskService`.

```java
ExampleServiceHost host = createAndStartExampleServiceHost();
TestRequestSender sender = new TestRequestSender(host);

// create a task
ExampleTaskServiceState postBody = new ExampleTaskServiceState();
postBody.documentSelfLink = "bar";
Operation post = Operation.createPost(host, "/core/example-tasks").setBody(postBody);
sender.sendAndWait(post, ExampleTaskServiceState.class);
```

Since it is a task service, it'll go through its workflow(state machine); so, we wait the task to finish.  
We use `TestContext#waitFor` method.
```java
// repeat given lambda until it returns true. timeout is set to 10sec.
waitFor(Duration.ofSeconds(10), () -> {
    Operation get = Operation.createGet(host, "/core/example-tasks/bar");
    ExampleTaskServiceState result = sender.sendAndWait(get, ExampleTaskServiceState.class);
    return isInFinalStage(result);
}, () -> "bar didn't reach the final stage");

...

private boolean isInFinalStage(ExampleTaskServiceState state) {
    EnumSet<TaskStage> finalStages = EnumSet.of(CANCELLED, FAILED, FINISHED);
    return state.taskInfo != null && finalStages.contains(state.taskInfo.stage);
}
```

Once wait logic has completed, the task has finished.  
Let's verify the finished task.

```java
Operation get = Operation.createGet(host, "/core/example-tasks/bar");
ExampleTaskServiceState result = sender.sendAndWait(get, ExampleTaskServiceState.class);
assertEquals(FINISHED, result.taskInfo.stage);
// verify more...
```


----

# Appendix

**Test Code**

```java
public class SampleTest {

    // use junit TemporaryFolder rule to create random working directory for ServiceHost
    @Rule
    public TemporaryFolder tempFolder = new TemporaryFolder();

    @Test
    public void verifyExampleService() throws Throwable {

        ExampleServiceHost host1 = createAndStartExampleServiceHost();
        ExampleServiceHost host2 = createAndStartExampleServiceHost();

        TestNodeGroupManager nodeGroup = new TestNodeGroupManager()
                .addHost(host1)
                .addHost(host2);

        nodeGroup.joinNodeGroupAndWaitForConvergence();  // join to default node group
        nodeGroup.waitForFactoryServiceAvailable("/core/examples");

        // create request sender using one of the hosts in the nodegroup
        TestRequestSender sender = new TestRequestSender(nodeGroup.getHost());

        ExampleServiceState postBody = new ExampleServiceState();
        postBody.name = "foo";
        postBody.counter = 0L;
        postBody.documentSelfLink = "foo";   // static self link: /core/examples/foo

        Operation post = Operation.createPost(host1, "/core/examples").setBody(postBody);

        // send POST and wait for response, then validate
        ExampleServiceState postResult = sender.sendAndWait(post, ExampleServiceState.class);
        assertEquals("foo", postResult.name);
        assertEquals("/core/examples/foo", postResult.documentSelfLink);


        List<Operation> patches = new ArrayList<>();
        for (int i = 0; i < 20; i++) {
            ExampleServiceState patchBody = new ExampleServiceState();
            patchBody.name = "foo-" + i;
            patchBody.counter = (long) (i + 1);  // 1-20
            Operation patch = Operation.createPatch(host1, "/core/examples/foo").setBody(patchBody);
            patches.add(patch);
        }
        // send 20 PATCH requests in parallel and wait all responses to be successful
        sender.sendAndWait(patches);

        Operation get = Operation.createGet(host1, "/core/examples/foo");
        ExampleServiceState getResult = sender.sendAndWait(get, ExampleServiceState.class);
        assertEquals(Long.valueOf(20), getResult.counter);

        // delete foo service
        Operation delete = Operation.createDelete(host1, "/core/examples/foo").setBody("{}");
        sender.sendAndWait(delete);

        // verify call to foo returns 404
        Operation getToDeleted = Operation.createGet(host1, "/core/examples/foo");
        FailureResponse failureResponse = sender.sendAndWaitFailure(getToDeleted);
        assertEquals(404, failureResponse.op.getStatusCode());
    }

    private ExampleServiceHost createAndStartExampleServiceHost() throws Throwable {
        Arguments args = new Arguments();
        args.sandbox = tempFolder.newFolder().toPath();  // use random directory
        args.port = 0;  // use random port

        ExampleServiceHost host = new ExampleServiceHost();
        host.initialize(args);
        host.start();
        return host;
    }

    @Test
    public void verifyExampleTaskService() throws Throwable {

        ExampleServiceHost host = createAndStartExampleServiceHost();
        TestRequestSender sender = new TestRequestSender(host);

        // create a task
        ExampleTaskServiceState postBody = new ExampleTaskServiceState();
        postBody.documentSelfLink = "bar";
        Operation post = Operation.createPost(host, "/core/example-tasks").setBody(postBody);
        sender.sendAndWait(post, ExampleTaskServiceState.class);

        // repeat given lambda until it returns true. timeout is set to 10sec.
        waitFor(Duration.ofSeconds(10), () -> {
            Operation get = Operation.createGet(host, "/core/example-tasks/bar");
            ExampleTaskServiceState result = sender.sendAndWait(get, ExampleTaskServiceState.class);
            return isInFinalStage(result);
        }, () -> "bar didn't reach the final stage");

        Operation get = Operation.createGet(host, "/core/example-tasks/bar");
        ExampleTaskServiceState result = sender.sendAndWait(get, ExampleTaskServiceState.class);
        assertEquals(FINISHED, result.taskInfo.stage);
        // verify more...
    }

    private boolean isInFinalStage(ExampleTaskServiceState state) {
        EnumSet<TaskStage> finalStages = EnumSet.of(CANCELLED, FAILED, FINISHED);
        return state.taskInfo != null && finalStages.contains(state.taskInfo.stage);
    }
}
```
