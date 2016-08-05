# Overview

The graph query task service provides  a task-based  REST API to  specify and
execute multi stage cross document queries, both  on the  node its
running, and across all nodes in  a specified node group. The service is
created through the service task factory (i.e., `/core/graph-query-tasks`).

## Prerequisites
Please first review the [query task tutorial](./QueryTaskService)

## Document graphs

Xenon services that use the **ServiceOption.PERSISTANCE** have each version of their state stored as a document in
the document index (which is replicated across nodes). Each service state document has a unique self link that
represents its primary key and its URI path in the REST API. A document graph is formed, when documents have fields
whose value is a link (or set of links) to some other document.

A document with different fields linking it to other documents:
```
public class ChildState extends ServiceDocument {
        @PropertyOptions(usage = { PropertyUsageOption.REQUIRED })
        public String name;
        @PropertyOptions(usage = { PropertyUsageOption.LINK })
        public String motherLink;
        @PropertyOptions(usage = { PropertyUsageOption.LINK })
        public String fatherLink;
        @PropertyOptions(usage = { PropertyUsageOption.LINK })
        public List<String> siblingLinks;
}

```

A common type of documents stored in xenon are configuration, or desired state documents.
Using examples from [photon model](https://github.com/vmware/photon-model)
consider a document used as a template (description) used to create compute instances with similar parameters.

An example of a [compute description](https://github.com/vmware/photon-model/blob/master/photon-model/src/main/java/com/vmware/photon/controller/model/resources/ComputeDescriptionService.java#L44)
```
GET /resources/compute-descriptions/vm-bronze
{
  "hostType": "VM_GUEST",
  "cpuCount": 1,
  "gpuCount": 0,
  "totalMemoryBytes": 2147483648,
  "environmentName": "AWS",
  "costPerMinute": 0.10,
  ....
  }
```

The description above is linked from a different document, the ComputeState that represent the actual compute instance. It has:
 * a **parentLink** - link to a parent (to represent nested relationships of a VM running inside a hypervisor host, or public cloud project)
 * a **descriptionLink** - link to the template used to create the compute instance
 * a list of **diskLinks** - links to documents describing attached storage
 * a list of **networkLinks** - links to documents describing attached networks
 
An example of a [ComputeState](https://github.com/vmware/photon-model/blob/master/photon-model/src/main/java/com/vmware/photon/controller/model/resources/ComputeService.java#L64)

```
GET /resources/compute/de9c3205-b898-4a22-9f4c-b520b0e5d16d

{
  "id": "c6ef4e54-2b76-43e4-bdf0-2c2520eede6c",
  "hostType": "VIRTUAL",
  "descriptionLink": "/resources/compute-descriptions/vm-bronze",
  "parentLink" : "/resources/compute/aws-endpoint-east",
  "diskLinks": [
    "/resources/disks/coreos-production"
  ],
  "powerState": "UNKNOWN",
  ....
  }
```

For detailed examples on graph queries, see the example section below.

# REST API

## URI Path

```
/core/graph-query-tasks

```

## POST

The GraphQueryTaskState must be specified in the operation body. If the
**taskInfo.isDirect** field is set to true, the POST will complete when
the task reaches a final state. Its recommended this is only enabled on
HTTP/2 aware clients and connections, or if the task is expected to execute quickly


### Task Specification

The task behavior is entirely driven by a a set of query stages reusing the
query specification from the [query task PODO](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/QueryTask.java).

The initial state must be an instance of [graph query task PODO](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/GraphQueryTask.java).

### Query stages

A graph query is composed out of 2 or more stages, where each stage selects a set of documents eligible as inputs for the next stage query specification.
The graph query requires the **QueryOption.SELECT_LINKS** option to be set
since it uses the selected link values to traverse the logical document graph.

Since the QueryTask PODO is re-used for the query specifications, most query options are supported (sorting, broadcast, etc)

### Pagination
For pagination examples, see the "tree graph with precomputed initial stage results" section below

### GraphQueryOption.FILTER_STAGE_RESULTS

When the FILTER_STAGE_RESULTS option is specified, the service will remove links from stages 0 -> N-1, that did not contribute to results in the final stage. In essence, this will leave the "parent" nodes that contributed to "child" results in the final stage, and all the intermediate nodes that are part of the parent->descendant directed graph 

### Multi node group graph queries
Note that the optional **nodeSelectorLink** field, per QueryTask stage specification can direct the query to execute against a specific set of xenon nodes, allowing a graph query to span node groups

# Examples

## Recursive tree graph (friends of friends)

The following demonstrates a 3 three stage graph query on a social graph like document graph. The query will find people in Washington State, then select which of their friends live in seattle, and finally, from all employers associated with those friends, which employers have an address in California

The query executes over an index that has documents of this type:
```
public class Person extends ServiceDocument {
        @PropertyOptions(usage = { PropertyUsageOption.REQUIRED })
        public String name;
        public String state;
        public String city;

        @PropertyOptions(usage = { PropertyUsageOption.LINK })
        public String employerLink;

        @PropertyOptions(usage = { PropertyUsageOption.LINK })
        public List<String> friendLinks;
}

public class Employer extends ServiceDocument {
        @PropertyOptions(usage = { PropertyUsageOption.REQUIRED })
        public String name;
        public String address;
}
```
The query stage specifications can be summarized as follows:
 * Stage zero - Filter by
   * **documentKind** equal to Person, AND
   * **state** equal Washington State AND
   * linkTerms = {"friendLinks"}
 * Stage one - Filter by
   * **city** equal Seattle
   * linkTerms = {"employerLink"}
 * Stage two - Filter by
   * **documentKind** equal to Employer, AND
   * **address** contains California

The code, in java, for building the query specification:
```
        QueryTask stageZero = QueryTask.Builder.create()
                .addLinkTerm("friendLinks")
                .setQuery(Query.Builder.create()
                        .addKindFieldClause(Person.class)
                        .addFieldClause("state", "Washington")
                        .build())
                .build();

      QueryTask stageOne = QueryTask.Builder.create()
                .addLinkTerm("employerLink")
                .setQuery(Query.Builder.create()
                        .addKindFieldClause(Person.class)
                        .addFieldClause("city", "Seattle")
                        .build())
                .build();

      QueryTask stageTwo = QueryTask.Builder.create()
                .setQuery(Query.Builder.create()
                        .addKindFieldClause(Employer.class)
                        .addFieldClause("state", "California")
                        .build())
                .build();

      GraphQueryTask initialGraphState = GraphQueryTask.Builder.create(3)
                .setDirect(true)
                .addQueryStage(stageZero)
                .addQueryStage(stageOne)
                .addQueryStage(stageTwo)
                .build();
      Operation createTask = Operation.createPost(this,ServiceUriPaths.CORE_GRAPH_QUERIES)
                .setBody(initialGraphState)
                .setCompletion((o,e) -> {.........})
                .sendWith(this);

```

The JSON serialized view of the initial graph task state, that kicks of the task:
```
{
"stages": [
    {
      "querySpec": {
        "query": {
          "occurance": "MUST_OCCUR",
          "booleanClauses": [
            {
              "occurance": "MUST_OCCUR",
              "term": {
                "propertyName": "state",
                "matchValue": "Washington State",
                "matchType": "TERM"
              }
            },
            {
              "occurance": "MUST_OCCUR",
              "term": {
                "propertyName": "documentKind",
                "matchValue": "com:vmware:xenon:services:samples:PersonService:Person",
                "matchType": "TERM"
              }
            }
          ]
        },
        "linkTerms": [
          {
            "propertyName": "friendLinks",
            "propertyType": "STRING"
          }
        ],
        "options": [
          "SELECT_LINKS"
        ]
      }
    },
   {
      "querySpec": {
        "query": {
          "occurance": "MUST_OCCUR",
          "booleanClauses": [
            {
              "occurance": "MUST_OCCUR",
              "term": {
                "propertyName": "city",
                "matchValue": "Seattle",
                "matchType": "TERM"
              }
            }
          ]
        },
        "linkTerms": [
          {
            "propertyName": "employerLink",
            "propertyType": "STRING"
          }
        ],
        "options": [
          "SELECT_LINKS"
        ]
      }
    },
    {
      "querySpec": {
        "query": {
          "occurance": "MUST_OCCUR",
          "booleanClauses": [
            {
              "occurance": "MUST_OCCUR",
              "term": {
                "propertyName": "address",
                "matchValue": "*California*",
                "matchType": "WILDCARD"
              }
            },
            {
              "occurance": "MUST_OCCUR",
              "term": {
                "propertyName": "documentKind",
                "matchValue": "com:vmware:xenon:services:samples:EmployerService:Employer",
                "matchType": "TERM"
              }
            }
          ]
        }
      },
      "indexLink": "/core/document-index",
      "nodeSelectorLink": "/core/node-selectors/default",
    },
``` 

