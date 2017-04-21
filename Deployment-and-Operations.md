# Deployment and Operations

This page is starting point for any deployment and post deployment related operations.

* [Single Node Deployment](Starting-Xenon-Host)
* [Multi-node Deployment](Starting-Xenon-Host.md#starting-multi-node-cluster)
   * [Examples](Starting-Xenon-Host.md#examples)
* [Blue/Green Upgrade](Side-by-Side-Upgrade)
   * [Continuous Migration](Side-by-Side-Upgrade.md#migrate-state-from-the-blue-cluster-to-the-green-cluster)
* [Backup and Restore](Backup-Restore)
* [Diagnosis and Troubleshooting](Debugging-and-Troubleshooting.md)
   * [Stats](Debugging-and-Troubleshooting.md#stats-per-service)
       * [Management Service Stats](HostManagementService.md#stats)
       * [Index Service Stats](Debugging-and-Troubleshooting.md#index-stats)
   * [Logs](Debugging-and-Troubleshooting.md#logging)
       * [Xenon Process logs](ServiceHostLogServiceDocumentation)
   * Node Group Health
       * [Node Group Basics](NodeGroupService.md)
       * [Node Group Convergence](NodeGroupService.md#node-group-convergence)
       * [Checking Node Group state](NodeGroupService.md#get)
       * [Dynamically Joining a Node Group](Multi-Node-Tutorial.md#dynamically-join-another-hosts-group-as-an-observer)
