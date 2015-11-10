# Overview

DCP framework is designed to build decentralized control and management plane components that are portable, lightweight (a few hundred bytes overhead per instance), fast, durable and observable. DCP is a CONTROL plane framework.

It is designed to address developer productivity and abstracts the choice of replication, durability and cluster management allowing us to choose the right tool, at the right time and for the right job. 

## How does DCP compare against GCE?

DCP is not comparable with GCE - a compute IaaS framework. Photon Controller (ESX Cloud) built on the control plane DCP is comparable to GCE if deployed on-prem or as-a-service.

## How does DCP compare against Mesosphere?

DCP is not comparable with Mesosphere. Mesosphere/DOCS and Mesos Scheduler/RM are functionality that would be built  on top of DCP, similar to what we are doing with ESX Cloud, Cell Manager, etc. 

## How does DCP compare against Zookeeper?

DCP is comparable to some extent with Zookeeper in that it helps with coordination and implementation of consensus. It differs from DCP in that it does not host code, does not support indexing, requires client libraries, etc. DCP developer do not need Zookeeper as DCP provides leader election, consensus and persistence as available (opt-in) features in its REST services (see design page and programming model for a list of service options) 
