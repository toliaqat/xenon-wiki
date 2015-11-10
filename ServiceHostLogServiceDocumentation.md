# Overview
The service host log service exposes the process logs as a structured JSON response through GET.

# Configuration

[java.util.logging](http://docs.oracle.com/javase/7/docs/api/java/util/logging/LogManager.html)
must be configured with a
[FileHandler](http://docs.oracle.com/javase/7/docs/api/java/util/logging/FileHandler.html), for example:

```
java.util.logging.FileHandler.pattern=/var/log/dcpHost.%g.log
```

The ServiceHost can be configured to use a FileHandler by pointing to a config file:

```
$ java -Djava.util.logging.config.file=contrib/dcpHostLogging.config -jar dcp-host/target/dcp-host-*-with-dependencies.jar
```

# REST API

## URI Path
```
core/management/process-log
```

## GET
The service defaults to returning every line in the current log file:

```
$ curl http://localhost:8000/core/management/process-log
```

Use the **lineCount** param to limit the number of log file lines:

```
$ curl http://localhost:8000/core/management/process-log?lineCount=5
{
  "items": [
    "[1][I][1411747104016][DecentralizedControlPlaneHost:8000][allocateExecutor][New executor for service /core/document-index]",
    "[2][I][1411747104179][ServiceHost][findListenPort][port candidate:58571]",
    "[3][I][1411747104181][ServiceHost][findListenPort][port candidate:58572]",
    "[4][I][1411747104182][8000/core/node-groups/default][startSerfAgentProcess][starting serf agent: /Users/dougm/vmw/dcp/bin/serf_darwin agent -bind\u003d0.0.0.0:58572 -rpc-addr\u003d127.0.0.1:58571 -node\u003dc6b0a327-ee75-4323-9063-fc916d55479a -tag /core/node-groups/default\u003dhttp://localhost:8000/core/node-groups/default]",
    "[5][W][1411747104509][8000/core/examples][getPeerChildServiceResults][(1735586309)Service has replication enabled but no group specified. Using default group]"
  ],
  "documentVersion": 0,
  "documentUpdateTimeMicrosUtc": 0
```

Use the **logFileNumber** param to get an older log file:

```
$ curl http://localhost:8000/core/management/process-log?logFileNumber=1
```

# Issues

Due to a bug in JRE8, you must manually remove the log .lck files after shutting down:

```
$ rm dcpHost.*.log.lck
```
