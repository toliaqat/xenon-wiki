## Overview
In this wiki we describe how to upgrade a Xenon based service using a
side-by-side approach.
Side-by-side in this case means that we will install the newer version of the
service on a separate set of nodes and migrate all necessary data from the old
node cluster to the new node cluster.

## Requirements
For the side-by-side upgrade to work your project will need to meet several
requirements:

* You need **additional resources** to set up a second node cluster.
* The new node cluster needs to be able to **connect** to the old node cluster.
* There might need to be a small time window where requests to the service are queued, at an edge device, or failed (the service appears offline).

## Assumptions
Besides the aforementioned requirements there is also a set of assumptions that
we are making:

* Only **data/state** needs to be migrated between the old and new node cluster.
* The system need to be tolerant to **temporarily inconsistent data**.
* On the old node cluster active **tasks will finish** during the maintenance
  window.
* **No indirect interaction** between old and new node cluster through third
  party entities, e.g. external data sources or hardware.
* Your **clients** will be able to connect to the new node cluster, e.g. you use
  an external load balancer point it to the new node cluster.

## Procedure
The side-by-side upgrade will proceed in the following steps:

1. Install the new service on a separate set of hardware.
* Enter the maintenance window.
* Do the data migration.
* Enable the new service.
* Exit maintenance window.

##### Entering The Maintenance Window
When entering the maintenance window users should not be able to start new task
on the old or new node cluster. Otherwise the data migration might not be
complete since the underlying Lucene query only returns results available before
the query was issue.

##### Data Migration
The data migration step starts one migration service for each data entity type
that needs to be migrated.
The migration service will query the old node cluster for entities and posts
them to the new node cluster.

In case your data model changed, e.g. new fields in the newer version of the
entity need to be filled in during migration you can supply a transformation
service that the MigrationService will call in order to transform the entity
before posting it to the destination.

This will allow you to do simple transformations like field re-namings or
filling in missing value as well as splitting and merging objects.
In order to merge objects the transformation service will need to call into the
old service to retrieve the necessary objects.


In case the amount of data is larger than you can possibly migrate within your
maintenance window you will need to start the migration before entering the
maintenance window.
To support continuous data migration the Migration service can be supplied with
a timestamp which it uses to query for all document updated after the time
stamp.
Also the migration service returns the latest document update time it saw while
paging through all entities it retrieved from the old service.

##### Enable The New Service
After the data migration is completed we need to make the service available to
all users.
This depends how you currently make the old service available to users.
In case you are using an external load-balancer you can point that load-blancer
to the new service.

##### Exiting The Maintenance Window
Now you can allow users to reach the new service.
