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

One example of service documents stored in xenon (in addition configuration, metrics, work flow tasks) is desired state,
documents that reflect the state of some real world entity. Using examples from [photon model](https://github.com/vmware/photon-model)
consider a document used as a template, a description used to create compute instances with similar parameters.

An example of a [compute description](https://github.com/vmware/photon-model/blob/master/photon-model/src/main/java/com/vmware/photon/controller/model/resources/ComputeDescriptionService.java#L44)
```
GET /resources/compute-descriptions/on-prem-vm-bronze
{
  "hostType": "VM_GUEST",
  "cpuCount": 1,
  "gpuCount": 0,
  "totalMemoryBytes": 2147483648,
  ....
  "environmentName": "AWS",
  "networkId": "vlan3201",
  "costPerMinute": 0.10,
  }
```

The description above is linked from a different document, the ComputeState that represent the actual compute instance. It has:
 * a **parentLink** - link to a parent (to represent nested relationships of a VM running inside a hypervisor host, or public cloud project)
 * a **descriptionLink** - link to the template used to create the compute instance
 
An example of a [ComputeState](https://github.com/vmware/photon-model/blob/master/photon-model/src/main/java/com/vmware/photon/controller/model/resources/ComputeService.java#L64)

```
GET /resources/compute/de9c3205-b898-4a22-9f4c-b520b0e5d16d

{
  "id": "c6ef4e54-2b76-43e4-bdf0-2c2520eede6c",
  "hostType": "VIRTUAL",
  "descriptionLink": "/resources/compute-descriptions/vm-bronze",
  "parentLink" : "/resources/compute/aws-endpoint-east",
  "address": "192.168.1.128",
  "diskLinks": [
    "/resources/disks/coreos-production"
  ],
  "powerState": "UNKNOWN",
  ....
  }
```

As you can see from the example instances of both documents above, we have now a simple document graph:

**Computes point to either descriptions or parents** (TODO Add diagram)

Some computes don't have parent links or descriptions, so they represent "islands" in the graph. Some descriptions are not referred to by any computes,
once again being isolated nodes that a graph traversal would never encounter.

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

A graph query is composed out of 2 or more stages, where each stage selects a s
The graph query requires the **QueryOption.SELECT_LINKS** option to be set
since it uses the selected link values to traverse the logical document graph.


# Examples

