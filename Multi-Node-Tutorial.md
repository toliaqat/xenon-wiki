# Overview

Xenon provides an operationally simple model to federate several nodes (Xenon service hosts) to form a node group. A node group provides high availability, scale out and the ability to dynamically expand / contract.
This tutorial will walk through starting a couple of xenon hosts, each in their process, have them join, and then demonstrate replication by adding a service instance on one node, then finding it on all nodes.

# Starting a host

In a terminal window, preferrably within a xenon enlistment, type the following to build the latest xenon jars:

`
cd xenon
git pull
mvn package -DskipTests=true
`
The above should have produced a host jar that we can now start in the terminal. This host will have just core services plus the example factory service (see example tutorial). Now lets start one host, on port 8000. If that port is not available on your system, use a different one.
`
java -jar xenon-host/target/xenon-host-0.5.1-SNAPSHOT-jar-with-dependencies.jar --port=8000 --peerNodes=http://127.0.0.1:8000,http://127.0.0.1:8001
`
At a different terminal, start a second host, on a different port, making sure you supply the proper port in the peerNodes argument:
`
java -jar xenon-host/target/xenon-host-0.5.1-SNAPSHOT-jar-with-dependencies.jar --port=8001 --peerNodes=http://127.0.0.1:8000,http://127.0.0.1:8001
`
Notice we started the second host with --port=8001

## Node join at startup

In xenon you can either dynamically join a node gorup by sending a POST to the local (un joined) node group service, or, you can have the xenon host join when it starts, by supplying the --peerNodes argument, with a list of URIs. To make startup scripts simple, you can supply the **same set of URIs, including the IP:PORT of the local host**. Xenon will ignore its self address but concurrently join through the other peer addresses.

When a node is starting you will see log output:
`
[0][I][1453320789994][DecentralizedControlPlaneHost:8000][startImpl][ServiceHost/2c6a499e listening on 127.0.0.1:8000]
[1][I][1453320789997][DecentralizedControlPlaneHost:8000][normalizePeerNodeList][Skipping peer http://127.0.0.1:8000, its us]
[2][I][1453320791195][DecentralizedControlPlaneHost:8000][lambda$4][Joined peer http://127.0.0.1:8001/core/node-groups/default]
[3][I][1453320791301][8000/core/node-groups/default][lambda$1][Synch quorum: 2. Sending POST to insert self (http://127.0.0.1:8000/core/node-groups/default) to peer http://127.0.0.1:8001/core/node-groups/default]
[4][I][1453320791302][8000/core/node-groups/default][handlePatch][Adding new peer 866041aa-e8a3-4cbf-ba49-8cf7c567f37d (http://127.0.0.1:8001/core/node-groups/default), status SYNCHRONIZING]
[5][I][1453320791302][8000/core/node-groups/default][handlePatch][State updated, merge with 866041aa-e8a3-4cbf-ba49-8cf7c567f37d, self 79418bdd-c8e4-4ce1-b98b-d073c9ab06c3, 1453320791301005]
[6][I][1453320797027][8000/core/node-groups/default][handlePatch][State updated, merge with 79418bdd-c8e4-4ce1-b98b-d073c9ab06c3, self 79418bdd-c8e4-4ce1-b98b-d073c9ab06c3, 1453320797027026]
