# Overview

In multi node environment, it is essential to perform certain logic only once in entire cluster.
Once the logic is executed, node group change such as adding/deleting/restarting nodes should NOT run the same logic again.  
This tutorial shows how to create one time execution service in entire cluster, calling it as a bootstrap service.

## Use Case

- Setup Admin user at system(nodegroup) startup
- Populate initial dataset
- Perform transform task only once
- etc

# Bootstrap Service

_code: [SampleBootstrapService.java](https://github.com/vmware/xenon/tree/master/xenon-samples/src/main/java/com/vmware/xenon/services/samples/SampleBootstrapService.java)_


```java
/**
 * Demonstrate one-time node group setup(bootstrap).
 *
 * This service is guaranteed to be performed only once within entire node group, in a consistent
 * safe way. Durable for restarting the owner node or even complete shutdown and restarting of all
 * nodes.
 */
public class SampleBootstrapService extends StatefulService {

    public static final String FACTORY_LINK = ServiceUriPaths.CORE + "/bootstrap";
    private static final String ADMIN_EMAIL = "admin@vmware.com";

    public static FactoryService createFactory() {
        return FactoryService.create(SampleBootstrapService.class);
    }

    /**
     * POST to a fixed self link that starts an initialization task.
     *
     * If the task is already in progress, or it has completed, the POST will be ignored.
     *
     * This method can be called at your {@link ServiceHost#start()} to trigger the task
     * at node start up.
     * The call will happen in every node start, but service options and implementation guarantees
     * actual logic will be performed only once within entire node group.
     *
     * Example how to call this method:
     * <pre>
     * {@code
     *     SampleBootstrapService.startTask(this, ServiceUriPaths.DEFAULT_NODE_SELECTOR,
     *                                        SampleBootstrapService.FACTORY_LINK);
     * }
     * </pre>
     *
     * @param host a service host
     * @param selectorPath node selector url path
     * @param factoryLink url path to check the service availability in node group
     */
    public static void startTask(ServiceHost host, String selectorPath, String factoryLink) {
        CompletionHandler ch = (o, e) -> {
            if (e != null) {
                // service is not yet available, reschedule
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e1) {
                    throw new IllegalStateException("interrupted", e1);
                }

                startTask(host, selectorPath, factoryLink);
                return;
            }

            // create service with fixed link
            ServiceDocument doc = new ServiceDocument();
            doc.documentSelfLink = "preparation-task";
            Operation.createPost(host, SampleBootstrapService.FACTORY_LINK)
                    .setBody(doc)
                    .setReferer(host.getUri())
                    .setCompletion((oo, ee) -> {
                        if (ee != null) {
                            host.log(Level.SEVERE, Utils.toString(ee));
                            return;
                        }
                        host.log(Level.INFO, "preparation-task triggered");
                    })
                    .sendWith(host);

        };

        NodeGroupUtils.checkServiceAvailability(ch, host, factoryLink, selectorPath);
    }

    public SampleBootstrapService() {
        super(ServiceDocument.class);
        toggleOption(ServiceOption.IDEMPOTENT_POST, true);
        toggleOption(ServiceOption.PERSISTENCE, true);
        toggleOption(ServiceOption.REPLICATION, true);
        toggleOption(ServiceOption.OWNER_SELECTION, true);
    }

    @Override
    public void handleStart(Operation post) {

        if (!ServiceHost.isServiceCreate(post)) {
            // do not perform bootstrap logic when the post is NOT from direct client, eg: node restart
            post.complete();
            return;
        }

        createAdminIfNotExist(getHost(), post);
    }

    @Override
    public void handlePut(Operation put) {

        if (put.hasPragmaDirective(Operation.PRAGMA_DIRECTIVE_POST_TO_PUT)) {
            // converted PUT due to IDEMPOTENT_POST option
            logInfo("Task has already started. Ignoring converted PUT.");
            put.complete();
            return;
        }

        // normal PUT is not supported
        put.fail(Operation.STATUS_CODE_BAD_METHOD);
    }

    private void createAdminIfNotExist(ServiceHost host, Operation post) {

        // Simple version of AuthorizationSetupHelper.
        // Prefer using TaskService for complex bootstrap logic.

        Query userQuery = Query.Builder.create()
                .addFieldClause(ServiceDocument.FIELD_NAME_KIND, Utils.buildKind(UserState.class))
                .addFieldClause(UserState.FIELD_NAME_EMAIL, ADMIN_EMAIL)
                .build();

        QueryTask queryTask = QueryTask.Builder.createDirectTask()
                .setQuery(userQuery)
                .build();

        URI queryTaskUri = UriUtils.buildUri(host, ServiceUriPaths.CORE_QUERY_TASKS);

        CompletionHandler userCreationCallback = (op, ex) -> {
            if (ex != null) {
                String msg = String.format("Could not make user %s: %s", ADMIN_EMAIL, ex);
                post.fail(new IllegalStateException(msg, ex));
                return;
            }

            host.log(Level.INFO, "User %s has been created", ADMIN_EMAIL);
            post.complete();
        };

        CompletionHandler queryUserCallback = (op, ex) -> {
            if (ex != null) {
                String msg = String.format("Could not query user %s: %s", ADMIN_EMAIL, ex);
                post.fail(new IllegalStateException(msg, ex));
                return;
            }

            QueryTask queryResponse = op.getBody(QueryTask.class);
            boolean userExists = queryResponse.results.documentLinks != null
                    && !queryResponse.results.documentLinks.isEmpty();

            if (userExists) {
                host.log(Level.INFO, "User %s already exists, skipping setup of user", ADMIN_EMAIL);
                post.complete();
                return;
            }

            // create user
            UserState user = new UserState();
            user.email = ADMIN_EMAIL;
            user.documentSelfLink = ADMIN_EMAIL;

            URI userFactoryUri = UriUtils.buildUri(host, ServiceUriPaths.CORE_AUTHZ_USERS);
            Operation.createPost(userFactoryUri)
                    .setBody(user)
                    .addRequestHeader(Operation.REPLICATION_QUORUM_HEADER,
                            Operation.REPLICATION_QUORUM_HEADER_VALUE_ALL)
                    .setCompletion(userCreationCallback)
                    .sendWith(this);
        };

        Operation.createPost(queryTaskUri)
                .setBody(queryTask)
                .setCompletion(queryUserCallback)
                .sendWith(this);

    }

}
```


## Service Options

```java
toggleOption(ServiceOption.IDEMPOTENT_POST, true);
toggleOption(ServiceOption.PERSISTENCE, true);
toggleOption(ServiceOption.REPLICATION, true);
toggleOption(ServiceOption.OWNER_SELECTION, true);
```

**IDEMPOTENT_POST**  
Convert POST request to PUT when service has already created. This is essential to make the service only created/executed once in entire cluster.

_Identifying service creation POST in `handleStart()`_  
`handleStart()` can be called in multiple situation such as node restart. (see more detail on javadoc)  
`ServiceHost.isServiceCreate(post)` can check whether the POST request is from a client to the factory for the service creation.

**PERSISTENCE**  

The service needs to be persisted.  
For example, the service information needs to be available after shutting down all nodes and starting them again. At the restart, it should NOT run the bootstrap logic because it has already performed before. Therefore, PERSISTENCE will keep the record for the past run and make sure the service will not invoked again.

## Triggering Bootstrap Service

POST request to the factory needs to be issued.
The triggering code will be usually placed in your `ServiceHost` implementation for starting the node.

For example, in `ExampleServiceHost`, `start()` can call `BootstrapService.performWhenReady` once all services are created.

```java
@Override
public ServiceHost start() throws Throwable {

    super.start();
    startDefaultCoreServicesSynchronously();

    // -snip- starting services...

    BootstrapService.performWhenReady(this, ServiceUriPaths.DEFAULT_NODE_SELECTOR, BootstrapService.FACTORY_LINK);

    return this;
}
```

When other nodes start up, it also calls `BootstrapService.performWhenReady()`.
Since the service is annotated as *IDEMPOTENT_POST*, the POST call will be converted to the PUT, then get discarded by the `handlePut()` implementation.
