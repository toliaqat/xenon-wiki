# Approach
A warm up run of the test is executed with same operation count, before the actual throughput test. 
In-process requests use the Kryo library for eagerly cloning the body at the send code site (avoiding serialization). Requests sent over sockets use the GSON JSON serialization library. 

Tests are run twice, in both case with the JVM heap constrained to a specified number (64MB and 2G).

Tests on the loopback interface are used to quantify the HTTP socket processing path (which uses the non blocking I/O APIs).

The framework provides an asynchronous HTTP client, with a connection pool. The connection pool flexes to N concurrent connections to the same service host, based on outbound operations, which naturally creates a multi client test case.

## Critical areas

The Xenon I/O pipeline has the following critical areas that require careful baseline and performance analysis before any changes
 * Indexing / Query - Please see the [lucene index service](LuceneDocumentIndexService) page
 * Message passing, in process - This page, for in memory, co-located service tests (stateful and stateless)
 * Network I/O - HTTP / HTTPS / HTTP2 / Websockets, all operations that are routed across nodes. This page
 * Replication - Related to network and in process message passing.

This [commit](https://github.com/vmware/xenon/commit/57e0b064a11760089520a55e571f124ffd415ce3) indicates the level of testing required for the 3 out of 4 categories above, see commit summary.

## Environment
### Software

* JVM - Oracle JRE 1.8 is used on Mac OSX Mavericks.
* OS System Version:	OS X 10.10.4 (14E46)
  *  Kernel Version:	Darwin 14.4.0
  *  Boot Volume:	SSD

### Hardware
2014 MacBook Pro Retina 15in
`
  Model Name:	MacBook Pro
  Model Identifier:	MacBookPro11,2
  Processor Name:	Intel Core i7
  Processor Speed:	2 GHz
  Number of Processors:	1
  Total Number of Cores:	4
  L2 Cache (per Core):	256 KB
  L3 Cache:	6 MB
  Memory:	16 GB
  Boot ROM Version:	MBP112.0138.B15
  SMC Version (system):	2.18f15
 
`

## Throughput analysis
It is strongly recommend to set the following JVM options when running performance tests or running production instances. To see peak performance, set the max and initial heap size for the JVM to something at least 512MB to 2G.

-Xmx4G

For constrained environment throughput examples use:

-Xmx64M
-Xms64M

### Service Footprint
 * ~ 400 bytes / service instance

Note Xenon will pause services to disk and remove all runtime cost, when a service is not in use. So its only limited by access patterns and disk size, not memory size.

## Running performance tests

The build verification tests used during maven build and test phases can also be used for performance analysis, by supplying various properties that modify service, request and other counts.

### In process client, in memory (not indexed) service
`
MAVEN_OPS="-Xmx8G" mvn test -Dtest=TestStatefulService#throughputInMemoryServicePut -Dxenon.requestCount=500000
`

### In process client, over OS sockets, HTTP+JSON, in memory (not indexed) service
`
MAVEN_OPS="-Xmx8G" mvn test -Dtest=NettyHttpServiceClientTest#throughputPutRemote -Dxenon.requestCount=1000 -Dxenon.serviceCount=128 -Dxenon.isStressTest=true
`
**Note** to set the property isStressTest=true, otherwise the test framework will timeout requests

### In process, 3 independent hosts, over OS sockets, replication with persistance, owner selection

`
MAVEN_OPS="-Xmx8G" mvn test -Dtest=TestNodeGroupService#replication -Dxenon.testDurationSeconds=1200 -Dxenon.totalOperationLimit=600000 -Dxenon.serviceCount=1000 -Dxenon.updateCount=33  -Dxenon.isStressTest=true
`

## Update Operation Throughput

### Single node
Using a payload that serializes to 636 bytes (JSON). For the durable service tests, throughput includes indexing cost, and commit to disk which occurs every 5 seconds.

The parameters supplied to the tests are serviceCount=128, and updateCount over 100,000 for in memory tests and over 10,000 for socket tests.

 * In memory service, in process (no socket I/O) (4G limit): 1,000,000 ops/sec
 * In memory service, in process (no socket I/O) (64MB limit): 500,000 ops/sec
 * In memory service, local sockets: 60,000 ops/sec
 * Durable service, in process (no socket I/O) (4G limit): 250,000 ops/sec
 * Durable service, in process (no socket I/O) (64MB limit): 50,000 ops/sec
The [lucene document index service](./luceneDocumentIndexService#performance) has more details on indexing and query throughput.

### Multiple node
 * Durable, replicated service, 3 nodes (4GB limit): 7,000 PUT ops/sec

The TestNodeGroupService.replication test method should be run with sufficiently large operation limit and test duration.
It logs the throughput, per action, at the end of the test:

`
[doReplication][Total operations: 630000]
[Total ops for POST: 9000, Throughput (ops/sec): 4155.109307]
[Total ops for PATCH: 297000, Throughput (ops/sec): 7587.162317]
[Total ops for PUT: 297000, Throughput (ops/sec): 8182.927047]
[Total ops for DELETE: 9000, Throughput (ops/sec): 6023.717383]
`

Detailed throughput numbers are available in the continuous integration tests in Jenkins.