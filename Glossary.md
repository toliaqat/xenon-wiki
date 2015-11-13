# Xenon Glossary Of Terms

### Micro Service
* A means of delivering value to customers by facilitating outcomes customers want to achieve without the ownership of specific costs and risks. 
* A collection of components, together with the policies that control its usage, organized to accomplish a specific function or set of functions. 

### Xenon Service
* A single software component, with a HTTP URI and a document describing it state. A service can use other services to offer simplified or advanced functionality. This is the building block of decentralized control plane nodes. 

#### Xenon Service Instance
A Xenon Service is defined a by a class (in Java, or a set of methods in Go)

* Each Service class may be instantiated one or more times.
* Each instance is identified and located via a unique path.

/core/examples/one
/core/examples/two

The above URI paths refer to two instances of the **example** service.

#### Xenon Service Instance Replica
Each instance is deployed over a group of Service Hosts (equivalently referred to as 'nodes'). One Service Host may end up hosting zero, or one replicas of the same Service Instance. At any time, a replica may be designated by the runtime as "owner". 

### Document
* A plain data object, a collection of properties that represent the state of a micro-service. A document can describe a long running task, or a document describing the health and operational status of an entire data center. Many services can use the same document type, but independent instances. Documents can be aggregated through queries.

### Node (Management Node)
* An instance of a service host process. A service host is a Java or Go process that hosts micro service instances. Its uniquely identified by a stable id and can be reached on a IP address plus TCP port. A single virtual or physical machine can have multiple nodes (multiple service host processes). We refer to nodes and service host interchangeably.

### Owner
* For a specific Service Instance, one of the replicas may be designated as owner. The precise guarantees regarding owner consistency and operations associated with it are described in the [replication and leader election page](leaderElectionAndReplicationDesignPage)

### Runtime
* The Xenon framework code, instantiated as a ServiceHost. The framework manages requests to services, service lifecycle, concurrency, statistics, etc. 

### Membership
* A dynamic state capturing the collection of nodes at some moment in time. Implemented by the [node group service](NodeGroupService)

### Membership View (or simply membership)
* A local value, held by a node, representing its knowledge of the membership

