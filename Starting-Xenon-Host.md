# Starting Xenon Host

## Xenon Host Command Line parameters

Following are the optional parameters that can be passed to Xenon host.

 * **--sandbox***=&lt;path&gt;*<br>
File directory path where all service documents are stored. By default a temporary directory is used. When the host is not running, you can delete the sandbox if you need to get back to "factory settings". When running application in docker container, this directory should be a persistent volume on the host machine to persist the data.<br>
**Example:**

        java -jar xenon-example.jar --sandbox=/data/xenon 


* **--peerNodes***=&lt;node list&gt;*<br>
Comma separated list of one or more peer nodes to join. Nodes must be defined in URI form, e.g `--peerNodes=http://192.168.1.59:8000,http://192.168.1.82`
You can either dynamically join a node group by sending a POST to the local (un-joined) node group service at /core/node-groups/default, or, you can have the xenon host join when it starts, by supplying the --peerNodes argument, with a list of URIs. To make startup scripts simple, you can supply the same set of URIs, including the IP:PORT of the local host. Xenon will ignore its self address but concurrently join through the other peer addresses. For more details on node group maintenance, see the [Node Group service](NodeGroupService).<br>
**Example:**

        java -jar xenon-example.jar --peerNodes=http://127.0.0.1:8000,http://127.0.0.1:8001 
When a node is starting you will see similar to following log output:



        [0][I][1453320789994][DecentralizedControlPlaneHost:8000][startImpl][ServiceHost/2c6a499e listening on 127.0.0.1:8000]
        [1][I][1453320789997][DecentralizedControlPlaneHost:8000][normalizePeerNodeList][Skipping peer http://127.0.0.1:8000, its us]
        [2][I][1453320791195][DecentralizedControlPlaneHost:8000][lambda$4][Joined peer http://127.0.0.1:8001/core/node-groups/default]



* **--id***=&lt;id&gt;*<br>
A stable identity associated with this host. Instead of hard to remember class 4 UUIDs, you can specify an ID when you start a Xenon host by providing the argument like `--id=hostAtPort8000`.

* **--publicUri***=&lt;uri&gt;*<br>
Optional public URI the host uses to advertise itself to peers. If its
not set, the bind address and port will be used to form the host URI. This option is best used when IP address of your host machine can change, and so instead of IP, a known hostname is used for communication with that machine. 

        java -jar xenon-example.jar \
             --publicUri=http://xenon-host1.vmware.com:8000 \
             --peerNodes=http://xenon-host1.vmware.com:8000,http://xenon-host2.vmware.com:8000,http://xenon-host3.vmware.com:8000


* **--port***=&lt;port&gt;*<br>
HTTP port to use for connection. The value should be a valid port number between 0 to 65536. But if `--securePort` option is used for secure transportation, then `--port` should be `-1`. 

* **--securePort***=&lt;port&gt;*<br>
HTTPS port to use for secure communication. If secure port is provided then `--port` argument should be `-1`

* **--sslClientAuthMode***=&lt;mode&gt;*<br>
SSL client authorization mode. Valid modes are `NONE`, `WANT` and `NEED`. Default is `NONE`. 

* **--keyFile***=&lt;file path&gt;*<br>
File path to key file (PKCS#8 private key file in PEM format).<br>
**Example**<br>

        --keyFile=./xenon-common/src/test/resources/ssl/server.pem
*Refer to [HTTPS for Network Communication](HTTPS-Network-Communication) for details on starting Xenon hosts with secure HTTPS capability.*

* **--keyPassphrase***=&lt;passphrase&gt;*<br>
Specify SSL private key passphrase.

* **--certificateFile***=&lt;file path&gt;*<br>
File path to certificate file. Since each Xenon Host acts as both, a Listener for incoming requests and a Client for making requests to other Xenon-hosts, it is important that the X.509 Certificate(s) used for the rest of the Xenon-Hosts, should be registered with either the default trust-store on the machine, or you can create a custom trust-store and use that at Host start up time.<br>
**Example**<br>

        --certificateFile=./xenon-common/src/test/resources/ssl/server.crt
*Refer to [HTTPS for Network Communication](HTTPS-Network-Communication) for details on starting Xenon hosts with secure HTTPS capability.*

* **--bindAddress***=&lt;address&gt;*<br>
Network interface address to bind to. By default Xenon will use `127.0.0.1` as bind address.

* **--perFactoryPeerSynchronizationLimitSeconds***=&lt;seconds&gt;*<br>
*[Advanced option]* An upper bound, in seconds, for service synchronization to complete. The runtime synchronizes
one replicated factory at a time. This limit applies to upper bound the runtime will wait for
a given factory, before moving on to the next. The factory that did not finish in time will stay
unavailable (/available will return error). The runtime will continue synchronization with the next
factory and the node will be marked as available even if one factory fails to complete in time.
If a factory does not finish in time, its availability can be explicitly reset with a PATCH to
the STAT_NAME_IS_AVALABLE, to the factory /stats utility service.<br>
A factory will accept POST requests, even during synchronization, and even if it fails to
complete synchronization in time. The availability indicator on /available is a hint, it does
not prevent the factory from functioning.<br>
The default value of 10 minutes allows for 1.8M services to synchronize, given an estimate of
3,000 service synchronizations per second, on a three node cluster, on a local network.<br>
Synchronization starts automatically if {@link Arguments#isPeerSynchronizationEnabled} is true,
and the node group has observed a node joining or leaving (becoming unavailable)

* **--isPeerSynchronizationEnabled***=&lt;bool&gt;*<br>
*[Advanced option]* Value indicating whether node group changes will automatically
trigger replicated service state synchronization. If set to false, client can issue
synchronization requests through core management service. Default is true.

* **--isAuthorizationEnabled***=&lt;bool&gt;*<br>
Mandate an auth context for all requests. This option will be set to true and authn/authz enabled by default after a transition period. Default is false.

* **--iauthProviderHostUri***=&lt;uri&gt;*<br>
Base URI of the xenon instance that acts as the auth source for this service host.

* **--resourceSandbox***=&lt;path&gt;*<br>
File directory path to resource files. If specified, resources will loaded from here instead of the JAR file of the host.

#### Passing command line arguments to the Xenon Host

Command line arguments can be passed to Xenon host by calling `initialize()` method of the ServiceHost.

## JVM Parameters

When starting Xenon based application, there are some JVM parameters you should tweak to adopt JVM to your application needs in production.

   * **_-Xmx_**&lt;heap size&gt;[g|m|k]<br>
Limit the maximum memory allocation pool for the JVM process. To avoid dynamic heap resizing and lags, we explicitly specify minimal and maximal heap size.

   * **_-Xms_**&lt;heap size&gt;[g|m|k]<br>
Allocate initial memory allocation pool for the JVM process. To avoid dynamic heap resizing and lags, we explicitly specify minimal and maximal heap size.

   * **_-XX:+UseConcMarkSweepGC_**<br>
> The Concurrent Mark Sweep (CMS) collector is designed for applications that prefer shorter garbage collection pauses and that can afford to share processor resources with the garbage collector while the application is running. Typically applications that have a relatively large set of long-lived data (a large tenured generation) and run on machines with two or more processors tend to benefit from the use of this collector. However, this collector should be considered for any application with a low pause time requirement. The CMS collector is enabled with the command-line option -XX:+UseConcMarkSweepGC.

   * **_-XX:MaxMetaspaceSize_**=&lt;metaspace size&gt;[g|m|k]<br>
Metaspace contains the Java objects associated with classes and interned strings. Metaspace size in Java VM 8 can grow without limits, but for the sake of system stability it makes sense to limit it with some finite value. 

   * **_-XX:+HeapDumpOnOutOfMemoryError_**<br>
   * **_-XX:HeapDumpPath_**=&lt;path to dump&gt;\`date\`.hprof<br>
If your application can crash because of out of memory error, then it is better to have above two options enabled to get heap dump for investigation instead of waiting for another repro. 

###Xenon properties
Following are some of the properties user can change in Xenon, passed through JVM.

   * **_-Djavax.net.ssl.trustStorePassword_**=\<password\><br>
Specify the password of the trust store being used.

   * **_-Djavax.net.ssl.trustStore_**=\<filepath\><br>
Specify the path to the trust store file(jks).

   * **_-Dxenon.NodeState.membershipQuorum_**=[n]<br>
[membershipQuorum](Multi-Node-Tutorial) is the minimum number of available nodes required for consensus operations and synchronization, we can specify the membershipQuorum by using a JVM xenon property `-Dxenon.NodeState.membershipQuorum=3`. 
  
   * **_-Dxenon.ServiceClient.DEFAULT_CONNECTIONS_PER_HOST_**=[n]<br>
Number of maximum parallel connections to a remote host. Idle connections are groomed but if this limit is set too high, and we are talking to many remote hosts, we can possibly exceed the process file descriptor limit. Default value is 128.

   * **_-Dxenon.FactoryService.MAX_SYNCH_RETRY_COUNT_**=[n]<br>
Maximum retries for child synchronization task. Default value is 8. We are using exponential backoff for synchronization retry, that means as the retry count increases the interval between retries will be increasing exponentially.

   * **_-Dxenon.FactoryService.SELF_QUERY_RESULT_LIMIT_**=[n]<br>
Maximum result limit used in querying the child services of a factory service for synchronization. Default value is 1000.

   * **_-Dxenon.SynchronizationTaskService.isDetailedLoggingEnabled_**=[bool]<br>
Property to enable detailed logging of child service synchronization task.

### Class Path
On java command we can specify the class path to start a particular host in Jar file. <br>
**Example**
```
java -cp xenon-example.jar com.vmware.xenon.services.common.ExampleServiceHost
```
In above example we are specifying the class path "com.vmware.xenon.services.common.ExampleServiceHost" to start the [ExampleServiceHost](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/ExampleServiceHost.java), which will start the factory service for creating ExampleTaskService. 

## OS Tweaks
Following are the suggested changes you would need to do in your relevant OS to run Xenon application in production.

* **Linux open file descriptors limit** <br>
By default most Linux distributions have very small file descriptor limit per process. Running your Xenon application for long duration might cause its process to run out of open file descriptors. Following is the command to change this limit to maximum limit before running Xenon application.

        ulimit -n `ulimit -Hn`

## Starting Multi-Node cluster
Following are the two most important arguments used for multi-node cluster of Xenon nodes. See the details of these command line arguments at above.
* --port
* --peerNodes

**Example**
Following example starts one host, on port 8000.
```
java \
  -jar xenon-host/target/xenon-host-*-jar-with-dependencies.jar \
  --port=8000 \
  --adminPassword=changeme \
  --peerNodes=http://127.0.0.1:8000,http://127.0.0.1:8001
```
At a different terminal, start a second host, on a different port, making sure you supply the proper port in the `--peerNodes` argument:
```
java \
  -jar xenon-host/target/xenon-host-*-jar-with-dependencies.jar \
  --port=8001 \
  --adminPassword=changeme \
  --peerNodes=http://127.0.0.1:8000,http://127.0.0.1:8001
```
Notice we started the second host with `--port=8001`

## Starting Xenon in Docker
When running Xenon application in a container, make sure you mount the Xenon sandbox directory to a persistent volume so that your data can persist between container restarts.

```bash
docker run -it \
           -v /data/xenon:/data/xenon \
           myXenonApp \
           java -jar myXenonApp-1.0.jar --sandbox=/data/xenon
```

Above example command mounts `/data/xenon` directory from host machine into the container and pass it as `sandbox` command line parameter to the Xenon java application.

*Please refer to [Docker Images](Docker-Images) page for more on Docker Images for Xenon.*

## Examples

----

**Example 1:**<br>
Start a Xenon host with HTTPS communication enabled by running the below command. More details at [HTTPS for Network Communication](HTTPS-Network-Communication).
```
java \
  -Djavax.net.ssl.trustStore=./xenon-common/src/test/resources/ssl/trustedcerts.jks \
  -Djavax.net.ssl.trustStorePassword=changeit \
  -jar xenon-host/target/xenon-host-*-with-dependencies.jar \
  --peerNodes=https://127.0.0.1:8000,https://127.0.0.1:8001 \
  --port=-1 --securePort=8000 \
  --adminPassword=changeme \
  --keyFile=./xenon-common/src/test/resources/ssl/server.pem \
  --certificateFile=./xenon-common/src/test/resources/ssl/server.crt
```

----

**Example 2:**<br>
Following three commands start three node cluster. We use three ports: 8000, 8001, 8002 for testing. Open three terminals and use the commands to start each node. See more details about this example at [Multi-Node Tutorial](Multi-Node-Tutorial).

```
java -Dxenon.NodeState.membershipQuorum=3 \
     -cp xenon-host/target/xenon-host-*-jar-with-dependencies.jar \
     com.vmware.xenon.services.common.ExampleServiceHost \
     --sandbox=/data/xenon --port=8000 \
     --adminPassword=changeme \
     --peerNodes=http://127.0.0.1:8000,http://127.0.0.1:8001,http://127.0.0.1:8002 --id=hostAtPort8000 

java -Dxenon.NodeState.membershipQuorum=3 \
     -cp xenon-host/target/xenon-host-*-jar-with-dependencies.jar \
     com.vmware.xenon.services.common.ExampleServiceHost \
     --sandbox=/data/xenon --port=8001 \
     --adminPassword=changeme \
     --peerNodes=http://127.0.0.1:8000,http://127.0.0.1:8001,http://127.0.0.1:8002 --id=hostAtPort8001

java -Dxenon.NodeState.membershipQuorum=3 \
     -cp xenon-host/target/xenon-host-*-jar-with-dependencies.jar \
     com.vmware.xenon.services.common.ExampleServiceHost \
     --sandbox=/data/xenon --port=8002 \
     --adminPassword=changeme \
     --peerNodes=http://127.0.0.1:8000,http://127.0.0.1:8001,http://127.0.0.1:8002 --id=hostAtPort8002
```

----

**Example 2:**<br>
This example shows many parameters used together including `--publicUri` that lets you specify a URI if the IP addresses are not the best way to configure hosts.

```
ulimit -n `ulimit -Hn`

java -Xmx512m -Xms512m \
     -XX:+UseConcMarkSweepGC \
     -XX:MaxMetaspaceSize=128m \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/dumps/xenon-app-\`date\`.hprof \
     -Dxenon.ServiceClient.DEFAULT_CONNECTIONS_PER_HOST=1000 \
     -Dxenon.SynchronizationTaskService.isDetailedLoggingEnabled=true \
     -Dxenon.NodeState.membershipQuorum=3 \
     -cp xenon-host/target/xenon-host-*-jar-with-dependencies.jar \
     com.vmware.xenon.services.common.ExampleServiceHost \
     --publicUri=http://xenon-host1.vmware.com:8000 \
     --sandbox=/data/xenon \
     --port=8000 \
     --id=xenon-host1 \
     --peerNodes=http://xenon-host1.vmware.com:8000,http://xenon-host2.vmware.com:8000,http://xenon-host3.vmware.com:8000
```

----

