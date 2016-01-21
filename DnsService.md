## Overview
The DNS service is built on top of Xenon. Other services can register and discover using a REST interface or the DNS interface. This is not a replacement or enhancement to  [Node Group Service](https://github.com/vmware/xenon/wiki/NodeGroupService). Both work together to simplify the deployment of a complex application.

## Xenon DNS Service

A Xenon DNS service contains key information of a set of services or nodes. You can have multiple services of the same type and behavior register with the DNS service. For example, a node group could host a set of nodes (N), each of which might contain one or more services. A client that needs to interact can communicate to any of them as Xenon routes the request to the appropriate node.

How does the client know where the nodes are hosted? They can be deployed in well known IP Addresses and port numbers. But this is not practical for larger deployments. Also, one should be able to deploy a node independently and then let it join an existing node group. But then how does a new node know at least one node in the group so that it can join? These are two primary motivations to provide the DNS service.

## DNS Participation

A service can participate by registering itself to the DNS service using a POST request. Clients and other services can discover registered services by a quick look-up. Two ways of discovering are 1) Issue a ODATA query filter 2) Perform a traditional DNS lookup ( A, SRV records , more on [Domain Name System] (https://en.wikipedia.org/wiki/Domain_Name_System)).

Note: Support to enable traditional DNS lookup is currently in progress.

During registration a service can specify a set of fields in the service record (described below). One of them is a health check URL and a lookup interval. The DNS service automatically marks the availability of the service.

## REST API

### URI
    /core/dns/service-records
    /core/dns/query

### GET


      curl http://localhost:8000/core/dns/query?$filter=serviceName%20eq%20ExampleFactoryService
      {
        "documentLinks": [
          "/core/dns/service-records/ExampleFactoryService"
        ],
        "documents": {
          "/core/dns/service-records/ExampleFactoryService": {
            "serviceLink": "/core/examples",
            "serviceName": "ExampleFactoryService",
            "healthCheckLink": "/core/examples/stats",
            "healthCheckIntervalSeconds": 1,
            "nodeReferences": [
              "http://127.0.0.1:62513",
              "http://127.0.0.1:62512",
              "http://127.0.0.1:62514"
            ],
            "serviceStatus": "AVAILABLE",
            "documentVersion": 7,
            "documentEpoch": 0,
            "documentKind": "com:vmware:xenon:dns:services:DNSService:DNSServiceState",
            "documentSelfLink": "/core/dns/service-records/ExampleFactoryService",
           ...
          }
        },
        ...
      }


### POST

A Xenon service or any REST service can by sending a POST to <DNSHost:port>/core/dns/service-records with the following body:


      {
        "serviceLink": "/core/examples",
        "serviceName": "ExampleFactoryService",
        "healthCheckLink": "/core/examples/stats",
        "healthCheckIntervalSeconds": 1,
        "tags": ["examples"],
        "nodeReferences": [
          "http://127.0.0.1:63012"],
        "documentSelfLink": "ExampleFactoryService"
      }


### Service Record

Each record in the DNS Service is represented with the DNSServiceState PODO

      public static class DNSServiceState extends ServiceDocument {
          public static final String FIELD_NAME_SERVICE_NAME = "serviceName";
          public static final String FIELD_NAME_SERVICE_TAGS = "tags";

          public enum ServiceStatus {
              AVAILABLE, UNKNOWN, UNAVAILABLE
          }

          @UsageOption(option = ServiceDocumentDescription.PropertyUsageOption.AUTO_MERGE_IF_NOT_NULL)
          public String serviceLink;
          @UsageOption(option = ServiceDocumentDescription.PropertyUsageOption.ID)
          public String serviceName;
          @UsageOption(option = ServiceDocumentDescription.PropertyUsageOption.OPTIONAL)
          public Set<String> tags;
          @UsageOption(option = ServiceDocumentDescription.PropertyUsageOption.AUTO_MERGE_IF_NOT_NULL)
          public String healthCheckLink;
          @UsageOption(option = ServiceDocumentDescription.PropertyUsageOption.AUTO_MERGE_IF_NOT_NULL)
          public Long healthCheckIntervalSeconds;
          @UsageOption(option = ServiceDocumentDescription.PropertyUsageOption.AUTO_MERGE_IF_NOT_NULL)
          public String hostName;
          @UsageOption(option = ServiceDocumentDescription.PropertyUsageOption.AUTO_MERGE_IF_NOT_NULL)
          public Set<URI> nodeReferences;
          @UsageOption(option = ServiceDocumentDescription.PropertyUsageOption.AUTO_MERGE_IF_NOT_NULL)
          public ServiceStatus serviceStatus;
      }
