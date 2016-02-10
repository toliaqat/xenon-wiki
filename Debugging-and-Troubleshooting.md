# Overview

Debugging a micro service, or a collection of micro services requires a certain approach. The Xenon system is 100% asynchronous and a micro service, when deployed, can be handling hundreds of thousands of operations. In addition, its state is indexed in the document store, has multiple version, and might not be cached in memory. Micro services run symmetrically across multiple nodes (service hosts) which creates further obstacles.

To address all of the above the Xenon system has been engineered to enable effective troubleshooting through the following mechanisms, some of them baked in the design itself.

In summary use the following mechanisms:
 * Interact with your service using HTTP clients and all the rich debugging support they offer (Browsers, curl, etc)
 * Take advantage of the multi-version, indexed state, per service, to observe the state evolution and all updates to your service(s)
 * Take advantage of per service <strong>stats</strong>! Add stats dynamically, within the service and then do GET on the /stats utility service, which listens on all INSTRUMENTED service instances
 * take advantage of the /core/management/process-log service, to see the logs, through HTTP
 * Use standard java logging through the JavaService.logXXXX() methods, or insert logs on one of the logging services so you can query any log event
 * Always have tests that run within a single machine, but using mocks and multiple instances of service hosts, simulate multi machine deployments. Take advantage of the fact Xenon can run 100s of hosts in one machine, with many services (millions each), all independent of each other.


## Clients
Xenon uses JSON and HTTP which means the web browser and curl are useful tools to interact with running services.

## Starting a host

The default host can be started as follows.
Navigate to the xenon-host directory

To start a host with default parameters, listening on port 8000:

```
java -jar xenon-host/target/xenon-host-*-with-dependencies.jar
```

To start a host listening on a particular interface address (useful for multi node) :

```
java -jar xenon-host/target/xenon-host-*-with-dependencies.jar --bindAddress=192.168.1.170
```

To start a host with a specific identifier (helps with multi-node debugging):

```
java -jar xenon-host/target/xenon-host-*-with-dependencies.jar --id=host-one
```


Replace ***192.168.1.170*** with a valid IP address on your node.

If you plan to join a peer node group do not use "localhost" as the address since Xenon currently only binds to one interface so it can deterministically advertise its public IP to peers.

You should see the following log messages in your terminal:
```
$ java -jar xenon-host/target/xenon-host-*-with-dependencies.jar --bindAddress=192.168.1.170

[1][I][1419530239289][ServiceHost:8000][loadState][loading previous state from /var/folders/ds/0173d2vj4zx9ty7vcdpg6bd80000gn/T/xenon/8000/serviceHostState.json]
[3][I][1419530239504][ServiceHost:8000][start][Listening on 192.168.1.170:8000]
[4][I][1419530239544][ServiceHost:8000][allocateExecutor][New executor for service /core/document-index]
```

If you want to experiment with replication, on the same node, in a different terminal, start a second host:

```
java -jar xenon-host/target/xenon-host-*-with-dependencies.jar --bindAddress=192.168.1.170 --port=8001 --peerNodes=http://192.168.1.170:8000
```

When the second host starts you should see join messages:

```
$ java -jar xenon-host/target/xenon-host-*-with-dependencies.jar --bindAddress=192.168.1.170 --port=8001 --peerNodes=http://192.168.1.170:8000

[3][I][1419530466530][ServiceHost:8001][getSystemInfo][Address fe80:0:0:0:4483:50ff:fe61:556a%awdl0 is unreachable, ignoring]
[4][I][1419530466663][ServiceHost:8001][start][Listening on 192.168.1.170:8001]
[5][I][1419530466717][ServiceHost:8001][allocateExecutor][New executor for service /core/document-index]
[8][I][1419530467026][ServiceHost:8001][lambda$2][Joined peer http://192.168.1.170:8000/core/node-groups/default]

```

## Client side tips

To interact with the default host, use a browser, curl, etc. For example, a GET on the core management service, on the local running host:
```
    curl http://localhost:8000/core/management
```

to get the entire process log for a remote Xenon host, do:

```
    curl http://192.168.1.19:8000/core/management/process-log
```

Or

```
dcpc get /core/management/go-dcp-process-log | jq -r .items[]
```

### Using the broadcast and P2P routing

Please see the node selector [forwarding REST API section](./NodeSelectorService#forwarding-service)

## Operation Tracing
To get a view of operations as they are passed throughout the system, Xenon supports [Operation Tracing](./OperationTracing).  The operation tracing service will index all inbound and outbound operations, so once again lucene can be used to build a complete timeline of all operations on the service host.

in summary:
 * enable operation tracing by sending a PATCH to /core/management
 * to see every single operation received by the host simple do a GET on the /core/operation-index with a wild card. For more advanced filters see the operation index page (use a query task)


```
GET /core/operation-index?documentSelfLink=*&expand=documentLinks

{
  "documentLinks": [
    "1440614086300002",
    "1440614086302006",
    "1440614086304006",
    ...
    ...
  ],
  "documents": {
    "1440614086306028": {
      "action": "POST",
      "host": "127.0.0.1",
      "port": 51696,
      "path": "/core/examples",
      "id": 119,
      "referer": "http://127.0.0.1:0/test-client-send",
      "jsonBody": "{\"keyValues\":{},\"counter\":14,\"name\":\"0x0000000e\",\"documentVersion\":0,\"documentUpdateTimeMicros\":0,\"documentExpirationTimeMicros\":0}",
      "statusCode": 0,
      "options": [],
      "documentVersion": 0,
      "documentKind": "com:vmware:xenon:common:Operation:SerializedOperation",
      "documentSelfLink": "1440614086306028",
      "documentUpdateTimeMicros": 1440614086306028,
      "documentExpirationTimeMicros": 1440700486306027
    },
    "1440614086329026": {
      "action": "POST",
      "host": "127.0.0.1",
      "port": 51696,
      "path": "/core/examples",
      "id": 487,
      "referer": "http://127.0.0.1:0/test-client-send",
      "jsonBody": "{\"keyValues\":{},\"counter\":76,\"name\":\"0x0000004c\",\"documentVersion\":0,\"documentUpdateTimeMicros\":0,\"documentExpirationTimeMicros\":0}",
      "statusCode": 0,
      "options": [],
      "documentVersion": 0,
      "documentKind": "com:vmware:xenon:common:Operation:SerializedOperation",
      "documentSelfLink": "1440614086329026",
      "documentUpdateTimeMicros": 1440614086329026,
      "documentExpirationTimeMicros": 1440700486329025
    },
...
...
}

```




## Querying the index
the document index service listens under /core/document-index and is used implicitly by the Xenon framework to manage indexed, durable state. It can be targeted with rich queries by creating a query task through the [query Task Service](./queryTaskService). You can also use it directly for simple prefix queries on document links, to find what services are started, essentially "walking" the URI tree.

The query below returns all services with a path starting with "4":
```
curl http://192.168.1.35:8000/core/document-index?documentSelfLink=/4*
{
  "documentLinks": [
    "/46b39b25-739f-428e-b2bd-f896ac10d2e8",
    "/499bd6b6-3ef8-4a8c-bf0c-6a4e5872a7f2",
    "/472ee5fb-dcf2-45bd-9b3e-ede0285a19db",
    "/483c873f-c45e-44cb-a1ec-80f49ebc3abe",
    "/4561451f-f01b-4caa-b2d9-6dbf68bcabfd",
    "/4c59994a-04b1-4159-9783-efccbbee0703"
    ....
```

This query returns all services with a path starting with "/core/examples/1":
```
url http://192.168.1.35:8000/core/document-index?documentSelfLink=/core/examples/4*
{
  "documentLinks": [
    "/core/examples/47e2cb46-5a15-4d71-a717-f0334fa5a37f",
    "/core/examples/45aa9760-107e-4532-ab23-3f4f854f107f",
    "/core/examples/4796e909-f0ab-42f2-8f12-7daf3a6fcdef",
    "/core/examples/44c1c3a8-2d49-41f2-b54b-f86ae5f50352",
    "/core/examples/4be340ee-969f-4f73-ae3e-01ac3e955426"
  ],
  "documents": {},
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "805d0d3f-0ed4-454c-8711-940fa1ebfb9a"
}
```

## State history
Durable micro services have their entire state evolution indexed. This means you can issue a query to document store micro service, using the service selflink as the "key" and retrieve all versions of a service state. This gives the developer a discrete time event history, which can be used to create a timeline of changes and a causal chain. With every state change Xenon also indexes the client URI/IP address and the authorization user.

### Index viewer (Lucene Luke)
Lucene comes with a nice index viewing tool that allows the developer to view what is in the index. The developer should download Luke and point it at the local file directory for the service host

# Logging
Xenon provides formatted logs that get included in the service host process log files under /var/log. If debugging a host locally, call the <strong>host.toggleDebugMode(true);</strong> method which will enable fine level logging for the host and all services, and delay operation expiration and maintenance.

Process logs are also available through the [log capture service](./ServiceHostLogServiceDocumentation) listening at /core/management/process-log

# Traffic capture for go and go-dcp within a container

To see traffic between dcp and go-dcp on CoreOS, use the [toolbox](https://coreos.com/docs/cluster-management/debugging/install-debugging-tools/)

Then watch the loopback interface for both ports:

```bash
tcpdump -i lo -s 0 -A port 8000 or port 8082
```


# Stats per service
Service stats can be enabled on-demand for services that do not have the INSTRUMENTED option. The following command for example will enable stats for the /core/document-index service:
```
curl -X PATCH -H "Content-Type: application/json" -d \  
'{ "addOptions": ["INSTRUMENTATION"], "kind": "com:vmware:xenon:common:ServiceConfigUpdateRequest" }' \  
"http://localhost:8000/core/document-index/config"
```
Please refer to the [REST API page](./REST-API#per-service-stats)

# Single machine development and debugging
Xenon service host and the micro service test framework enables, and requires, that all logic can be run on a single machine, using multiple service hosts. This allows the developer to test and debug decentralized, scale-out scenarios but with out requiring test VM or container provisioning. It also allows the developer to use a single IDE instance to debug any one of the service host processes that act as a standalone Xenon node.

The same code that runs across multiple hosts on machine should always run, un modified, across machines. Xenon is tolerant of latencies, unless the developer explicitly sets tight expirations per operation.

## Source code debugging
A modern IDE is recommended and test-driven approach should be used to develop code. Create a JUnit functional test that starts a service host (see multiple existing examples under testsrc/) and start one or more micro services. Use the IDE source code debugger and call host.toggleDebuggingMode(true) when developing your service. Make sure debugging mode is not enabled during regular test runs

# Memory leak analysis
The eclipse standalone (no need to use or install eclipse) [memory analyzer tool](http://www.eclipse.org/mat/downloads.php) can acquire heap data from a running process or analyze a heap binary exported through jmap. Use the Histogram feature, and limit the objects using the regex row at the top of the object column

# Mocking
Xenon services are HTTP endpoints. This means that complex third party services or other micro services can be mocked, and started as part of the test environment. This allows a functional test to be written without being deployed in a complex production-like environment.
