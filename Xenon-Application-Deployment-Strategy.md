_Working in progress_

----

## About this document

This document provides suggestions for how to setup xenon based application system in various use cases.

----

## Blue/Green deployment

TBD

see [[Side-by-Side-Upgrade]]

----

## Multiple Geo system

### Use Case
- cross datacenter replication (east coast, west coast)

TBD

----

## Active/Standby with different node group

### Scenario
Two node groups: ACTIVE and STANDBY.
ACTIVE group receives all requests.
STANDBY group receives data in near real time and both systems will have identical data.

### Use Case
- backup node group
- usage for read transactions such as data warehouse

### Recommendation

#### Continuous Migration
Use `MigrationTaskService` to periodically pull all data from ACTIVE to STANDBY.
`MigrationTaskService` has `continuousMigration` flag that will continuously query data and reflect to STANDBY node group.

#### Manual Replication
Subscribe to a service, then manually replicate to a service associated with the other node group (issue updates, in a eventual way)

----

## Multiple nodegroup system

### Scenario




----

## Read operation intensive system

### Scenario

### Use Case
- high traffic contents oriented website such as blogs

----
