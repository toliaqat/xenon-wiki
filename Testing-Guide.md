# Overview

This document explains how to write tests for xenon based application.

## Testing framework

Xenon comes with support for writing tests.

- `VerificationHost` : `ServiceHost` implementation with testing support methods
- `TestRequestSender` : provides methods to send request synchronously
- `TestContext` : counter and timer based wait condition
- `TestNodeGroupManager`: provides node group related operations
- `AuthTestUtils` : provides authorization related methods

All of these classes are packaged in `xenon-common-[version]-tests.jar`.

**Sample code**

- [TestExampleWithMultiNode][TestExampleWithMultiNode]
- [TestExampleService][TestExampleService]
- [TestExampleTaskService][TestExampleTaskService]

# Basics

## Making synchronous request

`TestRequestSender` provides means to send operations synchronously.

```java
TestRequestSender sender = new TestRequestSender(host);

// single request and wait the response
ExampleServiceState result = sender.sendAndWait(op, ExampleServiceState.class);

// multiple request in parallel, then wait all response to be returned
List<ExampleServiceState> results = sender.sendAndWait(ops, ExampleServiceState.class);

// expect failure response that contains callback operation and failure(Throwable)
FailureResponse failureResponse = sender.sendAndWaitFailure(op);
Operation callbackOp = failureResponse.op;
Throwable failure = failureResponse.failure;
assertEquals(403, callbackOp.getStatusCode());  // sample to check status code
```

## Waiting

To test async system, it is essential to properly put wait condition.  
`TestContext` provides count down and timeout style waiting for action(s) to be completed.


**Count down style wait:**

```java
// setting countdown=10
TestContext waitContext = new TestContext(10, Duration.ofSeconds(30));

// in async logic, for example completion handler
op.setComplate((o,e) -> {
  // ... some logic
  waitContext.complete();  // countdown
  // ...
});

// wait until complete() is called 10 times, or timeout
waitContext.await();
```

To make it simple for general wait case for countdown=1, `TestContext#waitFor` provides a boolean condition check callback with timeout.

**Timeout style wait:**

```java
waitFor(timeout, () -> {
    // check logic called multiple times
    ...
    return workDone;  // boolean whether it's done or not
}, () -> {
    // lambda to lazily construct timeout message
    return "Something was expected 100 but was " + actual;
});
```


If you are using `VerificationHost`, it has `waitFor` method as well.

If `TestContext` waiting mechanism is not enough for your use case, consider using 3rd party library such as [Awaitility][awaitility].


## Node group support

For testing with multiple nodes, you need to make a node group to cluster individual nodes.
`TestNodeGroupManager` provides operations for node group such as updating quorum, waiting node group convergence, or waiting OWNER_SELECTED service to be ready in the node group.

`TestNodeGroupManager` does not keep the state of target node group. It simply issues operations(requests) to registered nodes. Therefore, it is caller's responsibility that registered nodes are in correct state such as node availability, accessibility, etc.

```java
TestNodeGroupManager myGroup =new TestNodeGroupManager("myGroup")
                .addHost(host1)
                .addHost(host2)
                .addHost(host3)
                .createNodeGroup() // create myGroup
                .joinNodeGroupAndWaitForConvergence();  // make them join myGroup

// wait OWNER_SELECTED service to be available in cluster
myGroup.waitForFactoryServiceAvailable("/core/examples");

// create a sender using one of the host in node group
TestRequestSender sender = new TestRequestSender(myGroup.getHost());
sender.sendAndWait(...);

// change quorum
myGroup.updateQuorum(2);
```

## Authentication/Authorization

`AuthTestUtils` provides all auth related methods.  
You have to decide what operations needs be performed under which user(system or any other users).

```java
// execute given lambda operations under system auth context
AuthTestUtils.executeWithSystemAuthContext(nodeGroup, () -> {
    nodeGroup.joinNodeGroupAndWaitForConvergence();
    nodeGroup.waitForFactoryServiceAvailable("/core/examples");
});

// perform operations with system auth context declaratively
AuthTestUtils.setSystemAuthorizationContext(host);
...
AuthTestUtils.resetAuthorizationContext(host);

// login and subsequent operations will be performed as foo@vmware.com
AuthTestUtils.loginAndSetToken(nodeGroup, "foo@vmware.com", "password");


// login and get the auth token
String token = AuthTestUtils.login(nodeGroup, "foo@vmware.com", "password");

// logout
AuthTestUtils.logout(nodeGroup);
```

# How to write test

## Testing single Service

There are multiple ways to test single implementation of `Service`.
Simplest unit test would be instantiating target service manually and call `handleGet` method.
If you want to run the service within `ServiceHost`, you can start the service on `VerificationHost` and make a request using `TestRequestSender` for synchronous call.
If you want to use own `ServiceHost` implementation, you can still start the host, then interact with `TestRequestSender`. In this case, the test is more like integration/blackbox test.

## Testing ServiceHost

You can instantiate your `ServiceHost` implementation in test, and interact with it using `TestRequestSender` and `TestNodeGroupManager`.  
Even though it is a single node test, it is recommended to always write test as if it were multi nodes. Many of bugs surfaces in multi node environment.

### Single node testing

In single node test, you can simply instantiate your service host and make requests using `TestRequestSender`.  
We recommend start switching to multi node test as soon as possible since multi node is more close to the production environment, and many timing issues may show up in multi node. 

```java
// prepare my service host
ServiceHost host = new ...
host.start();

// prepare operation sender(client)
TestRequestSender sender = new TestRequestSender(host);

Operation post = ...
ExampleServiceState result = sender.sendAndWait(post, ExampleServiceState.class);
```

### Multiple node testing

```java
TestNodeGroupManager nodeGroup = new TestNodeGroupManager();
nodeGroup.addHost(host1);
nodeGroup.addHost(host2);

// make node group join to the "default" node group, then wait cluster to be stabilized
nodeGroup.joinNodeGroupAndWaitForConvergence();

// wait the service to be available in cluster
nodeGroup.waitForFactoryServiceAvailable("/core/examples");

// prepare operation sender using one of the host in the node group
ServiceHost peer = nodeGroup.getHost();
TestRequestSender sender = new TestRequestSender(peer);
```

See more example in [TestExampleWithMultiNode][TestExampleWithMultiNode].

## Authentication/Authorization

When `ServiceHost` has protected with auth(`host#setAuthorizationEnabled(true)`), it is caller's responsibility to properly setup authorization to the target operations.  
`AuthTestUtils` provides methods for login, logout, or set system auth context.  
So, use those methods appropriately to interact with protected resources.

See concrete example for `AuthTestUtils` usage in `authorization()` method in [TestExampleWithMultiNode][TestExampleWithMultiNode].



[awaitility]: https://github.com/awaitility/awaitility
[TestExampleWithMultiNode]: https://github.com/vmware/xenon/blob/master/xenon-common/src/test/java/com/vmware/xenon/services/common/TestExampleWithMultiNode.java

[TestExampleService]: https://github.com/vmware/xenon/blob/master/xenon-common/src/test/java/com/vmware/xenon/services/common/TestExampleService.java

[TestExampleTaskService]: https://github.com/vmware/xenon/blob/master/xenon-common/src/test/java/com/vmware/xenon/services/common/TestExampleTaskService.java
