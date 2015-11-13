# Approach
A warm up run of the test is executed with same operation count, before the actual throughput test. 
In-process requests use the Kryo library for eagerly cloning the body at the send code site (avoiding serialization). Requests sent over sockets use the GSON JSON serialization library. 

Tests are run twice, in both case with the JVM heap constrained to a specified number (64MB and 2G).

Tests on the loopback interface are used to quantify the HTTP socket processing path (which uses the non blocking I/O APIs).

THe framework provides an asynchronous HTTP client, with a connection pool. The connection pool flexes to N concurrent connections to the same service host, based on outbound operations, which naturally creates a multi client test case.

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

-Xmx2G
-XX:+UseParallelOldGC

For constrained environment throughput examples use:

-Xmx64M
-Xms64M

## Memory analysis
It is very challenging to get accurate per instance byte allocation counts. When using JVisualVM large service counts cause the JVM and VisualVM to hang. The most reliable alternative is to use the java runtime allocated and free memory number before and after the test. The JVM heap must be limited to get stable results:

-Xmx16M
-Xms16M

<strong>For tests that involve durability, increase working set to 64MB.</strong>

Make sure to use the ServiceTestEnvironment.snapshotAndLogMemoryInfo(instanceCount) to get the report at the end of the test method. It forces several GC collection rounds before computing deltas.

Using the constrained JVM heap size and Runtime.getRuntime().totalMemory / freeMemory APIs, N service instances are created, with at least one state update. <strong>The memory delta is calculated after several forced rounds of GC.</strong>

### Service Footprint
 * ~ 300 bytes / service instance

Note Xenon will pause services to disk and remove all runtime cost, when a service is not in use. So its only limited by access patterns and disk size, not memory size.

## Update Operation Throughput

### Single node
Using a payload that serializes to 524 bytes (JSON). For the durable service tests, throughput includes indexing cost, and commit to disk which occurs every 5 seconds.

 * In memory service, in process (no socket I/O) (2G limit): 1,000,000 ops/sec
 * In memory service, in process (no socket I/O) (64MB limit): 500,000 ops/sec
 * In memory service, local sockets: 30,000 ops/sec
 * Durable service, in process (no socket I/O) (512MB limit): 190,000 ops/sec
 * Durable service, in process (no socket I/O) (64MB limit): 50,000 ops/sec
 * Durable service, local sockets: 30,000 ops/sec

The [lucene document index service](luceneDocumentIndexService#performance) has more details on indexing and query throughput.

### Multiple nodes
 * Durable, replicated service, 5 nodes (2GB limit): 2,000 ops/sec
 
 GET / read throughput is higher for all permutations (for example, for durable services its about 4X write perf, on machine tested, since GETs execute in parallel from the multi version store)

Detailed throughput numbers are available in the continuous integration tests in Jenkins.

These are early numbers, we plan to optimize several parts of the I/O path, but as it stands, this is a usable system.
