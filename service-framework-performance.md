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

## CI/CD Performance numbers

Every xenon patch is verified for performance regressions using a [CI job](https://xn-jenkins-master.adlbg.eng.vmware.com/job/xenon-perf/) during pre-flight checks. Job runs in the (VMware internal) jenkins service, but an example output is included here:

```
18:15:33 +--------------------------------------------------------------+------------+------------+----------------------+
18:15:33 | Test/Metric                                                  |   Baseline |     Change |                Delta |
18:15:33 +--------------------------------------------------------------+------------+------------+----------------------+
18:15:33 | TestLuceneDocumentIndexService.throughputPut                 |            |            |                      |
18:15:33 |   * throughput:avg                                           |  226672.36 |  224571.47 |   -2100.89    -0.93% |
18:15:33 |   * throughput:max                                           |  327365.73 |  323232.32 |   -4133.41    -1.26% |
18:15:33 |   * throughput:min                                           |   69470.83 |   73457.68 |    3986.85    +5.74% |
18:15:33 | TestLuceneDocumentIndexService.throughputPost                |            |            |                      |
18:15:33 |   * POSTs/sec:avg                                            |   31613.74 |   30116.13 |   -1497.60    -4.74% |
18:15:33 |   * POSTs/sec:max                                            |  102728.47 |  105429.63 |    2701.16    +2.63% |
18:15:33 |   * POSTs/sec:min                                            |     126.27 |     129.87 |       3.60    +2.85% |
18:15:33 | TestLuceneDocumentIndexService.throughputSelfLinkQuery       |            |            |                      |
18:15:33 |   * GET /core/examples links/sec thpt (upd: false) :avg      |   55708.26 |   68499.04 |   12790.79   +22.96% |
18:15:33 |   * GET /core/examples links/sec thpt (upd: false) :max      |   55708.26 |   68499.04 |   12790.79   +22.96% |
18:15:33 |   * GET /core/examples links/sec thpt (upd: false) :min      |   55708.26 |   68499.04 |   12790.79   +22.96% |
18:15:33 |   * GET /core/examples links/sec thpt (upd: true) :avg       |  146841.71 |   95815.26 |  -51026.44   -34.75% |
18:15:33 |   * GET /core/examples links/sec thpt (upd: true) :max       |  146841.71 |   95815.26 |  -51026.44   -34.75% |
18:15:33 |   * GET /core/examples links/sec thpt (upd: true) :min       |  146841.71 |   95815.26 |  -51026.44   -34.75% |
18:15:33 |   * GET /immutable-examples links/sec thpt (upd: false) :avg |   22257.85 |   21297.56 |    -960.29    -4.31% |
18:15:33 |   * GET /immutable-examples links/sec thpt (upd: false) :max |   22257.85 |   21297.56 |    -960.29    -4.31% |
18:15:33 |   * GET /immutable-examples links/sec thpt (upd: false) :min |   22257.85 |   21297.56 |    -960.29    -4.31% |
18:15:33 |   * GET /immutable-examples links/sec thpt (upd: true) :avg  |   82054.57 |   66618.78 |  -15435.79   -18.81% |
18:15:33 |   * GET /immutable-examples links/sec thpt (upd: true) :max  |   82054.57 |   66618.78 |  -15435.79   -18.81% |
18:15:33 |   * GET /immutable-examples links/sec thpt (upd: true) :min  |   82054.57 |   66618.78 |  -15435.79   -18.81% |
18:15:33 |   * POSTs/sec:avg                                            |   12567.24 |   13427.67 |     860.43    +6.85% |
18:15:33 |   * POSTs/sec:max                                            |   15024.51 |   15524.97 |     500.46    +3.33% |
18:15:33 |   * POSTs/sec:min                                            |   10109.97 |   11330.36 |    1220.39   +12.07% |
18:15:33 | TestStatelessService.throughputPost                          |            |            |                      |
18:15:33 |   * throughput:val                                           |  512032.77 |  558503.21 |   46470.44    +9.08% |
18:15:33 | TestNodeGroupService.replication                             |            |            |                      |
18:15:33 |   * DELETE throughput ops/sec:avg                            |    6312.97 |    6156.99 |    -155.98    -2.47% |
18:15:33 |   * DELETE throughput ops/sec:max                            |    6312.97 |    6156.99 |    -155.98    -2.47% |
18:15:33 |   * DELETE throughput ops/sec:min                            |    6312.97 |    6156.99 |    -155.98    -2.47% |
18:15:33 |   * PATCH throughput ops/sec:avg                             |    7675.78 |    7842.32 |     166.53    +2.17% |
18:15:33 |   * PATCH throughput ops/sec:max                             |    7675.78 |    7842.32 |     166.53    +2.17% |
18:15:33 |   * PATCH throughput ops/sec:min                             |    7675.78 |    7842.32 |     166.53    +2.17% |
18:15:33 |   * POST throughput ops/sec:avg                              |    4457.33 |    5179.46 |     722.13   +16.20% |
18:15:33 |   * POST throughput ops/sec:max                              |    4457.33 |    5179.46 |     722.13   +16.20% |
18:15:33 |   * POST throughput ops/sec:min                              |    4457.33 |    5179.46 |     722.13   +16.20% |
18:15:33 |   * PUT throughput ops/sec:avg                               |    7674.51 |    7813.26 |     138.74    +1.81% |
18:15:33 |   * PUT throughput ops/sec:max                               |    7674.51 |    7813.26 |     138.74    +1.81% |
18:15:33 |   * PUT throughput ops/sec:min                               |    7674.51 |    7813.26 |     138.74    +1.81% |
18:15:33 | TestStatefulService.throughputInMemoryServicePut             |            |            |                      |
18:15:33 |   * throughput:avg                                           |  820452.40 |  846752.22 |   26299.82    +3.21% |
18:15:33 |   * throughput:max                                           |  868090.88 |  892296.97 |   24206.09    +2.79% |
18:15:33 |   * throughput:min                                           |  667709.96 |  684491.98 |   16782.02    +2.51% |
18:15:33 | NettyHttpServiceClientTest.throughputPutRemote               |            |            |                      |
18:15:33 |   * HTTP1.1 JSON PUTS/sec:avg                                |   36547.38 |   36314.96 |    -232.43    -0.64% |
18:15:33 |   * HTTP1.1 JSON PUTS/sec:max                                |   38823.17 |   38271.79 |    -551.38    -1.42% |
18:15:33 |   * HTTP1.1 JSON PUTS/sec:min                                |   30548.93 |   29871.65 |    -677.28    -2.22% |
18:15:33 |   * HTTP1.1 binary PUTS/sec:avg                              |   45601.41 |   46442.06 |     840.65    +1.84% |
18:15:33 |   * HTTP1.1 binary PUTS/sec:max                              |   46477.85 |   49951.22 |    3473.37    +7.47% |
18:15:33 |   * HTTP1.1 binary PUTS/sec:min                              |   44329.00 |   42595.67 |   -1733.33    -3.91% |
18:15:33 | TestNodeGroupService.forwardingAndSelection                  |            |            |                      |
18:15:33 |   * throughput:avg                                           | 2273345.87 | 2311881.27 |   38535.40    +1.70% |
18:15:33 |   * throughput:max                                           | 2601156.07 | 2639296.19 |   38140.12    +1.47% |
18:15:33 |   * throughput:min                                           | 1744186.05 | 1621621.62 | -122564.42    -7.03% |
18:15:33 | NettyHttp2Test.throughputPutRemote                           |            |            |                      |
18:15:33 |   * HTTP/2 JSON PUTS/sec:avg                                 |   26039.71 |   26134.88 |      95.17    +0.37% |
18:15:33 |   * HTTP/2 JSON PUTS/sec:max                                 |   27312.49 |   27553.55 |     241.05    +0.88% |
18:15:33 |   * HTTP/2 JSON PUTS/sec:min                                 |   21647.22 |   22268.62 |     621.40    +2.87% |
18:15:33 |   * HTTP/2 binary PUTS/sec:avg                               |   29392.55 |   29811.87 |     419.32    +1.43% |
18:15:33 |   * HTTP/2 binary PUTS/sec:max                               |   30906.68 |   31143.55 |     236.88    +0.77% |
18:15:33 |   * HTTP/2 binary PUTS/sec:min                               |   26887.93 |   28754.35 |    1866.42    +6.94% |
18:15:33 +--------------------------------------------------------------+------------+------------+----------------------+
18:15:33 Details for the first test:
18:15:33 {
18:15:33   "name": "TestLuceneDocumentIndexService.throughputPut",
18:15:33   "javaVersion": "1.8.0_102-b14",
18:15:33   "os": "Linux 3.19.0-74-generic",
18:15:33   "timestamp": "Wed Jan 18 17:56:58 UTC 2017",
18:15:33   "hardwareConfig": "desktop-jv",
18:15:33   "jvmArgs": [
18:15:33     "-XX:+HeapDumpOnOutOfMemoryError",
18:15:33     "-XX:+PrintGCDateStamps",
18:15:33     "-XX:+PrintGCDetails",
18:15:33     "-XX:+PrintGCTimeStamps",
18:15:33     "-XX:+UseG1GC",
18:15:33     "-XX:+UseGCLogFileRotation",
18:15:33     "-XX:G1HeapRegionSize=4m",
18:15:33     "-XX:GCLogFileSize=1m",
18:15:33     "-XX:HeapDumpPath=/home/jenkins/workspace/xenon-perf/hprof/481/xenon-perf-TestLuceneDocumentIndexService-throughputPut-%p.hprof",
18:15:33     "-XX:NumberOfGCLogFiles=10",
18:15:33     "-XX:OnOutOfMemoryError=kill -9 %p",
18:15:33     "-Xloggc:/home/jenkins/workspace/xenon-perf/hprof/481/xenon-perf-TestLuceneDocumentIndexService-throughputPut-%p.log",
18:15:33     "-Xmx8192m",
18:15:33     "-Xss1024k"
18:15:33   ],
18:15:33   "id": "af1c1b139e9cdbf410d7c80d37aa79895f1a56c6",
18:15:33   "metrics": {
18:15:33     "throughput:avg": 224571.46989065074,
18:15:33     "throughput:max": 323232.3232323232,
18:15:33     "throughput:min": 73457.67575322812
18:15:33   },
18:15:33   "stats": {
18:15:33     "/core/document-index": {}
18:15:33   },
18:15:33   "xenonArgs": {
18:15:33     "xenon.isStressTest": true,
18:15:33     "xenon.serviceCount": 128,
18:15:33     "xenon.updateCount": 2000,
18:15:33     "xenon.iterationCount": 10
18:15:33   }
18:15:33 }

```


## Environment
### Software

* JVM - Oracle JRE 1.8 is used on Mac OSX Mavericks.
* OS System Version:  OS X 10.10.4 (14E46)
  *  Kernel Version:  Darwin 14.4.0
  *  Boot Volume: SSD

### Hardware
2014 MacBook Pro Retina 15in
`
  Model Name: MacBook Pro
  Model Identifier: MacBookPro11,2
  Processor Name: Intel Core i7
  Processor Speed:  2 GHz
  Number of Processors: 1
  Total Number of Cores:  4
  L2 Cache (per Core):  256 KB
  L3 Cache: 6 MB
  Memory: 16 GB
  Boot ROM Version: MBP112.0138.B15
  SMC Version (system): 2.18f15
 
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

# Running performance tests

The build verification tests used during maven build and test phases can also be used for performance analysis, by supplying various properties that modify service, request and other counts.

Please **always specify the performance profile** otherwise maven default profile will restrict the JVM to 256M of heap

`
mvn test -DskipAnalysis=true -DskipGO -P performance -Dxenon.isStressTest=true
`

**Note** to set the property isStressTest=true, otherwise the test framework will timeout requests


## Single node
### Stateful service, in memory (not persisted)

#### in process client
`
mvn test -DskipAnalysis=true -DskipGO -P performance -Dtest=TestStatefulService#throughputInMemoryServicePut -Dxenon.serviceCount=64 -Dxenon.requestCount=50000 -Dxenon.isStressTest=true
`

#### In process client, over OS sockets, HTTP+JSON, HTTP+BINARY(KRYO)
`
mvn test -DskipAnalysis=true -DskipGO -P performance -Dtest=NettyHttpServiceClientTest#throughputPutRemote -Dxenon.requestCount=1000 -Dxenon.serviceCount=128 -Dxenon.isStressTest=true
`

## Multi node
### Stateful, persisted service, in process, 3 nodes, over OS sockets

`
mvn test -DskipAnalysis=true -DskipGO -P performance -Dtest=TestNodeGroupService#replication -Dxenon.testDurationSeconds=200 -Dxenon.totalOperationLimit=600000 -Dxenon.serviceCount=1000 -Dxenon.updateCount=33  -Dxenon.isStressTest=true
`

## Update Operation Throughput

### Single node
Using a payload that serializes to 330 bytes (JSON). For the durable service tests, throughput includes indexing cost, and commit to disk which occurs every 5 seconds.

The parameters supplied to the tests are serviceCount=128, and updateCount over 100,000 for in memory tests and over 10,000 for socket tests.

 * In memory service, in process (no socket I/O) (4G limit): 1,000,000 ops/sec
 * In memory service, in process (no socket I/O) (64MB limit): 500,000 ops/sec
 * In memory service, local sockets: 84,000 ops/sec
 * PUT Persisted service, in process (no socket I/O) (4G limit): 250,000 ops/sec
 * PUT Persisted service, in process (no socket I/O) (64MB limit): 50,000 ops/sec
 * POST (service creation) Persisted service, in process (no socket I/O) (4G limit): 60,000 ops/sec
 * POST (service creation) Persisted, Immutable service, in process (no socket I/O) (4G limit): 150,000 ops/sec

## Index/Query Throughput
The [lucene document index service](./luceneDocumentIndexService#performance) has more details on indexing and query throughput.

### Multiple node
 * Durable, replicated service, 3 nodes (4GB limit): 8,000 PUT ops/sec

The TestNodeGroupService.replication test method should be run with sufficiently large operation limit and test duration.
It logs the throughput, per action, at the end of the test:
`
[doReplication][Total operations: 630000]
[Total ops for POST: 9000, Throughput (ops/sec): 7032.109307]
[Total ops for PATCH: 297000, Throughput (ops/sec): 8082.162317]
[Total ops for PUT: 297000, Throughput (ops/sec): 8182.927047]
[Total ops for DELETE: 9000, Throughput (ops/sec): 6023.717383]
`

Detailed throughput numbers are available in the continuous integration tests in Jenkins.