# Overview

The lucene document index service provides document durability, indexing
and querying,  per Xenon host instance.  It abstracts the lucene  APIs and
exposes rich query functionality through  a "sister" service, the [query
task](./queryTaskService).

There is also a "blob" index, that can be used to store binary content and queried by a primary key. See the [blob index service](./luceneBlobIndexService) for details.

The document  index is a singleton  and does NOT do  replication between
nodes. Replication  is done when  updates are sent to  service instances
that are  marked DURABLE and  REPLICATED. The document index  service is
not  available to  external clients.  It is  only used  within each  dcp
process, as part of the service operation  work flow. It is not meant to
be used directly as a document store.

# REST API

## URI

```
/core/document-index
```

Note that updates to the index are only possible from service code 
running inside the same node as the lucene index service. Queries are 
allowed from remote services and clients.

## Performance

Please first read the [dcp performance page](./service-framework-performance) for environment setup. Tests below were run on the same environment as the Xenon framework tests. All performance tests come from the checked in tests, run with specific update count and service count parameters, plus JVM heap size limits.

### Write (indexing) throughput

Using TestLuceneDocumentIndexService#throughputPatch with serviceCount = 10000 and updateCount = 100.
***Throughput: 192K updates / sec***

```
[62][I][1437159809802][VerificationHost:55171][doServiceUpdates][Bytes per payload 524]
[63][I][1437159809802][VerificationHost:55171][testStart][Test doDurableServiceUpdate:doServiceUpdates, iterations 1000000, started]
[64][I][1437159811969][VerificationHost:55171][testWait][Test doServiceUpdates, iterations 1000000, waiting ...]
[65][I][1437159814986][VerificationHost:55171][testWait][Test doServiceUpdates, iterations 1000000, complete!]
[66][I][1437159814986][VerificationHost:55171][logThroughput][Test doServiceUpdates iterations per second: 192904.992941]
[67][I][1437159814986][VerificationHost:55171][logMemoryInfo][Memory free:360046000, available:1754791936, total:7635730432]
```

### Query Throughput
The   service  throughput   tests  measure   performance  in   terms  of
documents  indexed  per  second,  and  documents  searched  per  second.
The  service   tracks  detailed   stats  (available  at   runtime  under
/core/document-index/stats) providing  runtime information  on documents
indexed, fields indexed, latency, etc

Query throughput numbers can be determined on a host by simply running
- TestQueryTaskService simpleDocumentIndexingThroughput test with -Ddcp.isStressTest=true
- TestQueryTaskService complexDocumentIndexingAndQueryThroughput test with -Ddcp.isStressTest=true

### Single field, single result match 
Sample run from the complex document indexing test, 19 fields per document, 100,000 service documents indexed:
```
[35][I][1424206011001][VerificationHost:56030][log][Document count: 100000, Expected match count: 1, Documents / sec: 49554013.875124]
[36][I][1424206012238][VerificationHost:56030][log][Document count: 100000, Expected match count: 100000, Documents / sec: 80842.181510]
[37][I][1424206012241][VerificationHost:56030][log][Document count: 100000, Expected match count: 1, Documents / sec: 33333333.333333]
[38][I][1424206012244][VerificationHost:56030][log][Document count: 100000, Expected match count: 0, Documents / sec: 49776007.964161]
```

```
[57][I][1437160033782][VerificationHost:55208][logServiceStats][Stats for http://127.0.0.1:55208/core/document-index
	Count		Avg		Total			Name
	00000007	00038.29	0000268.00	commitDurationMicros
	00666792	00048.44	32297209.00	querySingleDurationMicros
	00000018	124944.00	2248992.00	resultProcessingDurationMicros
	00000017	16819.59	0285933.00	queryDurationMicros
	00000007	00001.00	0000007.00	commitCount
	00800000	00024.59	19671538.00	indexingDurationMicros
	00000001	08000.00	0008000.00	queryAllVersionsDurationMicros
	00800000	00008.63	6900000.00	indexedFieldCount
	00000014	00001.00	0000014.00	indexSearcherUpdateCount
	00000007	381864.86	2673054.00	indexedDocumentCount
	00000007	00001.00	0000007.00	maintenanceCount
	00800000	00008.63	6900000.00	fieldCountPerDocument
```
Note that performance becomes O(n) when large number of results satisfy the query. It means retrieving each document and expanding its content.

### Self link match query, 100K results

When result count matches document count, throughput decreases since we have to post process and filter each result and reject expired documents, older versions, etc.


```
[Document count: 100000, Expected match count: 100000, Documents / sec: 78125.000000]
[Document count: 100000, Expected match count: 100000, Documents / sec: 174520.069808]
[Document count: 100000, Expected match count: 100000, Documents / sec: 188679.601282]
[Document count: 100000, Expected match count: 100000, Documents / sec: 170940.463146]
[Document count: 100000, Expected match count: 100000, Documents / sec: 190839.694656]
```

### Boolean query with numeric range clause, single result

Sample result on same machine, running a boolean query against 200K documents, same field count:
```
{
  "occurance": "MUST_OCCUR",
  "booleanClauses": [
    {
      "occurance": "MUST_OCCUR",
      "term": {
        "propertyName": "documentKind",
        "matchValue": "com:vmware:dcp:services:common:QueryValidationTestService:QueryValidationServiceState"
      }
    },
    {
      "occurance": "MUST_OCCUR",
      "term": {
        "propertyName": "doubleValue",
        "matchType": "TERM",
        "range": {
          "type": "DOUBLE",
          "min": 123.2,
          "max": 123.21,
          "isMinInclusive": true,
          "isMaxInclusive": false,
          "precisionStep": 4
        }
      }
    }
  ]
}]
```

Throughput for above query:
```
[Document count: 200000, Expected match count: 0, Documents / sec: 1954094.318564]
```

### Stats
See the stats section below for gathering detailed latency information.


# REST Api

## URI Path
```
/core/document-index
```
## Verbs

### GET
Retrieves document content given a selflink or self link mask. Complex queries are only available through the query task service, not directly through this service


### DELETE
Does an orderly shutdown of the service

### GET on /stats

```
{
  "entries": {
    "commitDurationMicros": {
      "name": "commitDurationMicros",
      "latestValue": 9.0,
      "accumulatedValue": 5313041.0,
      "version": 16,
      "lastUpdateMicrosUtc": 1424205513203012
    },
    "querySingleDurationMicros": {
      "name": "querySingleDurationMicros",
      "latestValue": 2.0,
      "accumulatedValue": 7996770.0,
      "version": 65592,
      "lastUpdateMicrosUtc": 1424205450953062,
      "logHistogram": {
        "bins": [
          57338,
          4545,
          3387,
          198,
          121,
          3,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0
        ]
      }
    },
    "resultProcessingDurationMicros": {
      "name": "resultProcessingDurationMicros",
      "latestValue": 1.0,
      "accumulatedValue": 6453013.0,
      "version": 24,
      "lastUpdateMicrosUtc": 1424205469558004,
      "logHistogram": {
        "bins": [
          18,
          0,
          1,
          0,
          0,
          0,
          5,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0
        ]
      }
    },
    "queryDurationMicros": {
      "name": "queryDurationMicros",
      "latestValue": 994.0,
      "accumulatedValue": 513933.0,
      "version": 23,
      "lastUpdateMicrosUtc": 1424205469558003,
      "logHistogram": {
        "bins": [
          6,
          0,
          8,
          1,
          7,
          1,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0
        ]
      }
    },
    "commitCount": {
      "name": "commitCount",
      "latestValue": 16.0,
      "accumulatedValue": 0.0,
      "version": 16,
      "lastUpdateMicrosUtc": 1424205513203010
    },
    "indexingDurationMicros": {
      "name": "indexingDurationMicros",
      "latestValue": 2.0,
      "accumulatedValue": 2.6317716E7,
      "version": 200000,
      "lastUpdateMicrosUtc": 1424205461263016,
      "logHistogram": {
        "bins": [
          178538,
          8855,
          11936,
          518,
          137,
          14,
          2,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0
        ]
      }
    },
    "queryAllVersionsDurationMicros": {
      "name": "queryAllVersionsDurationMicros",
      "latestValue": 16999.0,
      "accumulatedValue": 16999.0,
      "version": 1,
      "lastUpdateMicrosUtc": 1424205427918003,
      "logHistogram": {
        "bins": [
          0,
          0,
          0,
          0,
          1,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0
        ]
      }
    },
    "indexedFieldCount": {
      "name": "indexedFieldCount",
      "latestValue": 2800000.0,
      "accumulatedValue": 0.0,
      "version": 200000,
      "lastUpdateMicrosUtc": 1424205461263017
    },
    "indexSearcherUpdateCount": {
      "name": "indexSearcherUpdateCount",
      "latestValue": 6.0,
      "accumulatedValue": 0.0,
      "version": 6,
      "lastUpdateMicrosUtc": 1424205461595000
    },
    "indexedDocumentCount": {
      "name": "indexedDocumentCount",
      "latestValue": 200000.0,
      "accumulatedValue": 0.0,
      "version": 200000,
      "lastUpdateMicrosUtc": 1424205461263019
    },
    "maintenanceCount": {
      "name": "maintenanceCount",
      "latestValue": 16.0,
      "accumulatedValue": 0.0,
      "version": 16,
      "lastUpdateMicrosUtc": 1424205513203000
    },
    "fieldCountPerDocument": {
      "name": "fieldCountPerDocument",
      "latestValue": 20.0,
      "accumulatedValue": 2800000.0,
      "version": 200000,
      "lastUpdateMicrosUtc": 1424205461263018,
      "logHistogram": {
        "bins": [
          100000,
          100000,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0,
          0
        ]
      }
    }
  },
  "documentVersion": 0,
  "documentKind": "com:vmware:dcp:common:ServiceStats",
  "documentSelfLink": "/core/document-index/stats",
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0
}
```
