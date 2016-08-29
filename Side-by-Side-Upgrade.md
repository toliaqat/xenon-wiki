## Overview
In this wiki we describe how to upgrade a Xenon based service using a
side-by-side approach.
Side-by-side in this case means that we will install the newer version of the
service on a separate set of nodes and migrate all necessary data from the old
node cluster to the new node cluster.

## Requirements
For the side-by-side upgrade to work your project will need to meet several
requirements:

* You need **additional resources** to set up a second node cluster.
* The new node cluster needs to be able to **connect** to the old node cluster.
* The new node cluster should be fully formed and stable. Its recommended quorum is set to *node group size** so node failures are not masked during upgrade and migration task fails (it can always be retried)
* There might need to be a small time window where requests to the service are queued, at an edge device, or failed (the service appears offline).

## Assumptions
Besides the aforementioned requirements there is also a set of assumptions that
we are making:

* Only **data/state** needs to be migrated between the old and new node cluster.
* The system need to be tolerant to **temporarily inconsistent data**.
* On the old node cluster active **tasks will finish** during the maintenance
  window.
* **No indirect interaction** between old and new node cluster through third
  party entities, e.g. external data sources or hardware.
* Your **clients** will be able to connect to the new node cluster, e.g. you use
  an external load balancer point it to the new node cluster.

#### Authentication/Authorization

If your cluster has enabled AuthN/AuthX services, in order to seamlessly access both nodes, same user(`documentSelfLink`) needs to exist in both node groups.
For example, if `example@vmware.com` user with `/core/authz/user-groups/example@vmware.com` documentSelfLink exists in old node group, then same user `/core/authz/user-groups/example@vmware.com` needs to exists in new node group.
As long as user documentSelfLinks are same, auth token for the user works for both node groups.


## Procedure
The side-by-side upgrade will proceed in the following steps:

1. Install the new service on a separate set of hardware.
* Enter the maintenance window.
* Do the data migration.
* Enable the new service.
* Exit maintenance window.

##### Entering The Maintenance Window
When entering the maintenance window users should not be able to start new task
on the old or new node cluster. Otherwise the data migration might not be
complete since the underlying Lucene query only returns results available before
the query was issue.

##### Data Migration
The data migration step starts one migration service for each data entity type
that needs to be migrated.
The migration service will query the old node cluster for entities and posts
them to the new node cluster.

In case your data model changed, e.g. new fields in the newer version of the
entity need to be filled in during migration you can supply a transformation
service that the MigrationService will call in order to transform the entity
before posting it to the destination.

This will allow you to do simple transformations like field re-namings or
filling in missing value as well as splitting and merging objects.
In order to merge objects the transformation service will need to call into the
old service to retrieve the necessary objects.


In case the amount of data is larger than you can possibly migrate within your
maintenance window you will need to start the migration before entering the
maintenance window.
To support continuous data migration the Migration service can be supplied with
a timestamp which it uses to query for all document updated after the time
stamp.
Also the migration service returns the latest document update time it saw while
paging through all entities it retrieved from the old service.

##### Enable The New Service
After the data migration is completed we need to make the service available to
all users.
This depends how you currently make the old service available to users.
In case you are using an external load-balancer you can point that load-blancer
to the new service.

##### Exiting The Maintenance Window
Now you can allow users to reach the new service.

## End to End Example
This walkthrough provides a hands-on example showcasing how Xenon supports a live, side-by-side upgrade of a service. For this walkthrough, the existing (old) Xenon service is running on what we'll call the "blue" node cluster; the new version of the Xenon service will be on a separate "green" node cluster.

> This walkthrough focuses solely on the Xenon components of a live upgrade; it will not go into specifics relating to a load balancer or API gateway.

### Configure Xenon Host to start TaskMigrationService
The [MigrationTaskService](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/MigrationTaskService.java) provides built-in support to migrate state from the "blue" node cluster to the new "green" node cluster.

Assuming you've cloned the `xenon` repo locally, modify the [DecentralizedControlPlaneHost.java](https://github.com/vmware/xenon/blob/master/xenon-host/src/main/java/com/vmware/xenon/host/DecentralizedControlPlaneHost.java)'s `start()` to also start the `MigrationTaskService`. This is done by adding the following line after the `UiService` is started.

```java
    // Start migration service
    super.startFactory(new MigrationTaskService());
    
    // Don't forget to add the import!
    // import com.vmware.xenon.services.common.MigrationTaskService;
```

### Startup both "blue" and "green" node clusters
Next, build Xenon locally using `mvn clean install -DskipTests`. Then from the root `xenon` directory, startup our "blue" node-cluster (which will only consist of a single Xenon node for this walkthrough):

```sh
java -jar xenon-host/target/xenon-host-0.8.0-SNAPSHOT-jar-with-dependencies.jar --port=8000 --id=blueNodeAtPort8000 --sandbox=xenon-host/target/xenonSandboxBlue
```

In a separate terminal window, startup the "green" node cluster in a similar manner (but on a different port).

```sh
java -jar xenon-host/target/xenon-host-0.8.0-SNAPSHOT-jar-with-dependencies.jar --port=8001 --id=greenNodeAtPort8001 --sandbox=xenon-host/target/xenonSandboxGreen
```

> NOTE: In our example, the [ExampleService](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/ExampleService.java) implementation is exactly the same between both "blue" and "green" deployments. In reality, these implementations would be different for an upgrade, but this walkthrough still shows how state from a previous node cluster can be migrated to a new node cluster.

> Also, we only have a single node in each "cluster" ... but this is intentional to keep the walkthrough straightforward. The same steps apply for a true "cluster" of nodes (given that the "blue" and "green" node clusters are separate from each other).

### Prepopulate "blue" node cluster with test data
Let's put in some test data into the "blue" node cluster. We'll use Apache's [ab](https://httpd.apache.org/docs/2.4/programs/ab.html) benchmarking utility to send `HTTP POST`s to create 1,000 example service instances, but feel free to use whatever tool you want.

```sh
echo '{"name": "example-1", "counter": 1}' > /tmp/xenonExampleService.json
ab -p /tmp/xenonExampleService.json -T "application/json" -c 5 -n 1000 http://localhost:8000/core/examples
```

At this point, the "blue" node has 1,000 example service instances; the "green" node has zero. You can verify this:

```sh
# 8000 --> blue, has 1000 instances ; 8001 --> green, has 0 instances
curl localhost:8000/core/examples
curl localhost:8001/core/examples
```

### Migrate state from the "blue" cluster to the "green" cluster

For this walkthrough, we only wish to migrate `ExampleService` state to the "green" cluster. As mentioned earlier, you'll need to create a `MigrationTaskService` instance for each service factory you wish to migrate.

Save the following `MigrationTaskService` details into a file located in `/tmp/xenonMigrateExampleService.json`:

```json
{
    "sourceNodeGroupReference": "http://localhost:8000/core/node-groups/default",
    "destinationNodeGroupReference": "http://localhost:8001/core/node-groups/default",
    "sourceFactoryLink": "/core/examples",
    "destinationFactoryLink": "/core/examples",
    "continuousMigration": "false"
}
```

> NOTE: `continuousMigration` is an optional field that defaults to `false`. If you set it to `true`, an ongoing migration task will be used (which defaults to firing every minute). See [MigrationTaskService](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/MigrationTaskService.java) for more details.

Fire off a `HTTP POST` to create the migration task in your favorite HTTP client.

```sh
curl -H "Content-Type: application/json" --data @/tmp/xenonMigrateExampleService.json http://localhost:8001/management/migration-tasks 
```

The response should supply a `documentSelfLink` where you can see the status of the migration. I used that value and confirmed the migration finished by examining the response of: `curl localhost:8001/management/migration-tasks/26167d31-63df-4dfc-a604-5774348d25f5`.

Now, you can also see that all the `ExampleService` state was migrated to the "green" cluster.

```sh
curl localhost:8001/core/examples
```

> It's important to note that the `MigrationTaskService` can compare a ServiceDocument's `documentUpdateTimeMicros` and only migrate it if it hasn't been migrated yet. This means you can kick off a new `MigrationTaskService` multiple times without worrying that duplicates will be added to the "green" cluster.

You're done! All the state from the "blue" cluster has successfully been migrated to the "green" cluster.

### "Upgrading" Data/State
The `MigrationTaskService` also supports changes to a service's State (aka: `ServiceDocument`) between versions. This comes in handy if the "green" version of your service introduces new fields (or renames fields) that need to be properly initialized from the old version's state.

The `MigrationTaskService` accepts a stateless `transformationServiceLink` that takes the "blue" (or old) state as an input, and transforms it to the expected "green" (or new) state when migrating.

In this walkthrough, we did not include a `transformationServiceLink` in our `POST` body, so there was no transformation... but this functionality is available if you need it.

### Selectively Choosing Data to Migrate
If you don't want to migrate **all** state to your "green" cluster, you can also provide a `querySpec` for your `MigrationTaskService` instance that  defines precisely the state that should be migrated.

If the migration task is migrating the `/core/examples` factory, the default `querySpec` used (if not provided) would be:
```
"querySpec": {
    "query": {
        "occurance": "MUST_OCCUR",
        "booleanClauses": [
        {
            "occurance": "MUST_OCCUR",
            "term": {
                "propertyName": "documentSelfLink",
                "matchValue": "/core/examples/*",
                "matchType": "WILDCARD"
            }
        } ]
    },
    "resultLimit": 500,
    "options": [
        "EXPAND_CONTENT"
    ]
}
```

More details on Xenon queries can be found:
* [Introduction to Service Queries](./Introduction-to-Service-Queries) 
* [Query Task Service Tutorial](./QueryTaskService)

### Concluding Thoughts

Admittedly, there is a bit of hand-waving here relating to a **truly live** upgrade with zero downtime, especially when it comes to how/when to configure an external load balancer or API gateway to ensure that your service's state is both consistent and available.

One method of ensuring a "live upgrade" might consist of:

1. Fire off migration task for each service factory you want to migrate
2. Monitor migration tasks. Once migration tasks are finished (or close to finished), configure your load balancer queue all requests
3. Perform a final migration and wait until all tasks finished
4. Configure load balancer to route traffic to "blue" node cluster

Even if the "blue" cluster contains a lot of data, the migration is done while the "blue" cluster is still live and responding to clients. Once **most** of the state is migrated, the maintanence window (where the load balancer is queuing requests) will be so small that clients will barely notice.