# Overview
This tutorial covers topics relating to multi node deployments, and the concepts behind high availability, leader election, and service restart during node group changes. It demonstrates the ease of developing robust, scale out workflows, modeled as task services, and the service options, best practices needed.

# Prerequisites

While replicated services deployed across multiple nodes should be the default type of service, this is an advanced topic since it requires some understanding of distributed system concepts, and the xenon programming model. Please make sure to read the tutorials below and *their* prerequisites

* [Example Tutorial](./Example-Service-Tutorial)
* [Multi Node Tutorial](./Multi-Node-Tutorial)
* [Xenon Clustering deck](https://github.com/vmware/xenon/blob/master/contrib/docs/XenonClustering.pptx)

# Key Concepts

Xenon relies on two mechanisms to understand and properly implement the needs of a service:

 * Service options - ServiceOption.OWNER_SELECTION and ServiceOption.REPLICATION
 * Node group configuration - Membership quorum

Service instances run on each node but the consensus protocol routes requests to a single node, the one elected as owner, for a particular service.

Using the same underlying protocol the runtime will guarantee that
 * updates happen atomically, service handlers execute only on service instance on owner node
 * periodic handleMaintenance only executes on owner node
 * handleStart only executes on owner node

The service author does not need to know which node is owner. It simply authors logic assuming it only runs on one node, and if nodes come and go, a new node will dynamically take over.

leader election and high availability come "embedded" in the programming model: since the update, maintenance and start handlers are guaranteed to run on one node, out of N, the author can initiate logic assuming the peers will see state changes and persist them, but will not duplicate the effort.