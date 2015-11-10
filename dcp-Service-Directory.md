This page serves as a hub for micro service documentation. A micro service
should be described in terms of the function it performs, interaction diagrams
with other services, and its REST API (its document and actions on the
document).

This directory can also be thought of as a functional component directory: Each
micro service serves a re-usable function (node membership, authz, RBAC,
analytics, monitoring, provisioning, document indexing, etc) across levels of
the system

# Service Directory

 * [Example Factory Service](dcp-Example-Service-Tutorial) - A factory service that creates simple document services allowing anyone to interact with DCP service hosts and experience all the DCP features in action (durability, indexing, complex query support, replication across nodes, REST API)
 * [Lucene Document Index Service](luceneDocumentIndexService) - Deeply indexes on disk, any state updates on a co-located service. It enables high throughput, complex queries across documents. Its local only, replication is done by the framework independently.
 * [Lucene Blob Index Service](luceneBlobIndexService) - Stores and retrieves binary content using a primary key.
 * [Query Task Service](QueryTaskService) - A task based service to process both local, and multi-node potentially long running queries over all indexed documents. A factory service, QueryTaskFactoryService, is used to create tasks that specify the query to execute and reflect the query progress. The service talks with the document index service, co-located in the same process and translates the query description to the lucene query format
 * [Node Group Service](NodeGroupService) - Tracks node membership using a scalable gossip layer.
 * [Node Selector Service](NodeSelectorService) - Selects nodes, given a node group and a key, using a consistent hashing algorithm. Custom node selector implementations can be supplied.
 * [Host Management Service](HostManagementService) - A simple REST front end to each DCP micro service host providing system information and means to change micro service host configuration
 * [Service Host Log service](ServiceHostLogServiceDocumentation) - Returns the process logs for the service host as a JSON response
 * [Dynamic service loader](LoaderService) - A service that reflects JAR files dropped in a directory under the sandbox, and loads any singleton services it finds, in the running service host.
 * [JavaScript Services](JavaScript-Services) - A mini-framework which allows to create DCP services hosted by client browser.