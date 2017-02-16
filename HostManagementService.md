# Overview

The service host management service exposes a REST api to manage the lifecycle of the service host, perform maintenance actions and retrieve detailed status.  
This service API also takes PATCH requests to perform various management related operations such as configure operation tracing request, backup/restore, and schedule peer synchronization.

# REST Api

## URI Path
```
/core/management
```
## Verbs

### GET
Retrieves status and state.
```
{
  "bindAddress": "0.0.0.0",
  "httpPort": 8000,
  "maintenanceIntervalMicros": 30000000,
  "operationTimeoutMicros": 30000000,
  "storageSandbox": "file:/tmp/dcp/8000/",
  "id": "65e5488f-72be-4d85-98da-d838ad2afa5d",
  "isStarted": true,
  "systemInfo": {
    "properties": {
      "gopherProxySet": "false",
      "awt.toolkit": "sun.lwawt.macosx.LWCToolkit",
      "file.encoding.pkg": "sun.io",
      "java.specification.version": "1.8",
      "sun.cpu.isalist": "",
      "sun.jnu.encoding": "UTF-8",
      "java.class.path": "/Users/georgec/VMware/dcp/target/test-classes:/Users/georgec/VMware/dcp/target/classes:/Users/georgec/.m2/repository/junit/junit/3.8.1/junit-3.8.1.jar:/Users/georgec/.m2/repository/com/google/code/gson/gson/2.3/gson-2.3.jar:/Users/georgec/.m2/repository/org/javassist/javassist/3.18.2-GA/javassist-3.18.2-GA.jar:/Users/georgec/.m2/repository/com/esotericsoftware/kryo/kryo/2.24.0/kryo-2.24.0.jar:/Users/georgec/.m2/repository/com/esotericsoftware/minlog/minlog/1.2/minlog-1.2.jar:/Users/georgec/.m2/repository/org/objenesis/objenesis/2.1/objenesis-2.1.jar:/Users/georgec/.m2/repository/org/apache/lucene/lucene-analyzers-common/4.9.0/lucene-analyzers-common-4.9.0.jar:/Users/georgec/.m2/repository/org/apache/lucene/lucene-core/4.9.0/lucene-core-4.9.0.jar:/Users/georgec/.m2/repository/org/apache/lucene/lucene-misc/4.9.0/lucene-misc-4.9.0.jar:/Users/georgec/.m2/repository/org/apache/lucene/lucene-queries/4.9.0/lucene-queries-4.9.0.jar:/Users/georgec/.m2/repository/org/msgpack/msgpack/0.6.11/msgpack-0.6.11.jar:/Users/georgec/.m2/repository/com/googlecode/json-simple/json-simple/1.1.1/json-simple-1.1.1.jar",
      "java.vm.vendor": "Oracle Corporation",
      "sun.arch.data.model": "64",
      "java.vendor.url": "http://java.oracle.com/",
      "user.timezone": "America/Los_Angeles",
      "os.name": "Mac OS X",
      "java.vm.specification.version": "1.8",
      "user.country": "US",
      "sun.java.launcher": "SUN_STANDARD",
      "sun.boot.library.path": "/Library/Java/JavaVirtualMachines/jdk1.8.0_11.jdk/Contents/Home/jre/lib",
      "sun.java.command": "com.vmware.dcp.services.common.DecentralizedControlPlaneHost --port\u003d8000 --bindAddress\u003d0.0.0.0",
      "http.nonProxyHosts": "local|*.local|169.254/16|*.169.254/16",
      "sun.cpu.endian": "little",
      "user.home": "/Users/georgec",
      "user.language": "en",
      "java.specification.vendor": "Oracle Corporation",
      "java.home": "/Library/Java/JavaVirtualMachines/jdk1.8.0_11.jdk/Contents/Home/jre",
      "file.separator": "/",
      "line.separator": "\n",
      "java.vm.specification.vendor": "Oracle Corporation",
      "java.specification.name": "Java Platform API Specification",
      "java.awt.graphicsenv": "sun.awt.CGraphicsEnvironment",
      "sun.boot.class.path": "/Library/Java/JavaVirtualMachines/jdk1.8.0_11.jdk/Contents/Home/jre/lib/resources.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_11.jdk/Contents/Home/jre/lib/rt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_11.jdk/Contents/Home/jre/lib/sunrsasign.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_11.jdk/Contents/Home/jre/lib/jsse.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_11.jdk/Contents/Home/jre/lib/jce.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_11.jdk/Contents/Home/jre/lib/charsets.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_11.jdk/Contents/Home/jre/lib/jfr.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_11.jdk/Contents/Home/jre/classes",
      "sun.management.compiler": "HotSpot 64-Bit Tiered Compilers",
      "ftp.nonProxyHosts": "local|*.local|169.254/16|*.169.254/16",
      "java.runtime.version": "1.8.0_11-b12",
      "user.name": "georgec",
      "path.separator": ":",
      "os.version": "10.9.5",
      "java.endorsed.dirs": "/Library/Java/JavaVirtualMachines/jdk1.8.0_11.jdk/Contents/Home/jre/lib/endorsed",
      "java.runtime.name": "Java(TM) SE Runtime Environment",
      "file.encoding": "UTF-8",
      "java.vm.name": "Java HotSpot(TM) 64-Bit Server VM",
      "java.vendor.url.bug": "http://bugreport.sun.com/bugreport/",
      "java.io.tmpdir": "/var/folders/7x/_539ncd577jgfflj33qsg3_w0000gn/T/",
      "java.version": "1.8.0_11",
      "user.dir": "/Users/georgec/VMware/dcp",
      "os.arch": "x86_64",
      "java.vm.specification.name": "Java Virtual Machine Specification",
      "java.awt.printerjob": "sun.lwawt.macosx.CPrinterJob",
      "sun.os.patch.level": "unknown",
      "java.library.path": "/Users/georgec/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:.",
      "java.vm.info": "mixed mode",
      "java.vendor": "Oracle Corporation",
      "java.vm.version": "25.11-b03",
      "java.ext.dirs": "/Users/georgec/Library/Java/Extensions:/Library/Java/JavaVirtualMachines/jdk1.8.0_11.jdk/Contents/Home/jre/lib/ext:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java",
      "sun.io.unicode.encoding": "UnicodeBig",
      "java.class.version": "52.0",
      "socksNonProxyHosts": "local|*.local|169.254/16|*.169.254/16"
    },
    "environmentVariables": {
      "PATH": "/usr/bin:/bin:/usr/sbin:/sbin",
      "SHELL": "/bin/bash",
      "APP_ICON_18288": "../Resources/Eclipse.icns",
      "JAVA_STARTED_ON_FIRST_THREAD_18288": "1",
      "USER": "georgec",
      "TMPDIR": "/var/folders/7x/_539ncd577jgfflj33qsg3_w0000gn/T/",
      "SSH_AUTH_SOCK": "/tmp/launch-yo0C96/Listeners",
      "JAVA_MAIN_CLASS_27558": "com.vmware.dcp.services.common.DecentralizedControlPlaneHost",
      "__CF_USER_TEXT_ENCODING": "0x1F5:0:0",
      "Apple_PubSub_Socket_Render": "/tmp/launch-tbKOgr/Render",
      "__CHECKFIX1436934": "1",
      "LOGNAME": "georgec",
      "HOME": "/Users/georgec"
    },
    "availableProcessorCount": 8,
    "freeMemoryByteCount": 276973280,
    "totalMemoryByteCount": 285736960,
    "maxMemoryByteCount": 3817865216,
    "ipAddresses": [
      "fc00:10:118:97:24dc:387d:461d:3c3f",
      "fc00:10:118:97:e4d:e9ff:fe99:26ec",
      "fe80:0:0:0:e4d:e9ff:fe99:26ec%en3",
      "10.118.97.165"
    ]
  },
  "lastMaintenanceTimeUtcMicros": 0,
  "pendingMaintenanceOperations": 0,
  "isProcessOwner": true,
  "documentVersion": 0,
  "documentUpdateTimeMicrosUtc": 0
}

```

### DELETE
Does an orderly shutdown of the host and all the services 

## Stats

The core management service reports important time series stats on key metrics:
 * CPU usage
 * Disk usage
 * Thread count
aggregated per hour, per minute, in rolling time series.

```
GET /core/management/stats
{
.....
"cpuUsagePercentPerDay": {
      "name": "cpuUsagePercentPerDay",
      "latestValue": 0.01548839632318137,
      "accumulatedValue": 116.30780660032818,
      "version": 141,
      "lastUpdateMicrosUtc": 1470417944799007,
      "kind": "com:vmware:xenon:common:ServiceStats:ServiceStat",
      "timeSeriesStats": {
        "bins": {
          "1470416400000": {
            "avg": 0.8248780609952354,
            "count": 141.0
          }
        },
        "numBins": 24,
        "binDurationMillis": 3600000,
        "aggregationType": [
          "AVG"
        ]
      }
    },
    "availableDiskBytesPerHour": {
      "name": "availableDiskBytesPerHour",
      "latestValue": 1.26101041152E11,
      "accumulatedValue": 1.7906572627968E13,
      "version": 142,
      "lastUpdateMicrosUtc": 1470417944799002,
      "kind": "com:vmware:xenon:common:ServiceStats:ServiceStat",
      "timeSeriesStats": {
        "bins": {
          "1470417720000": {
            "avg": 1.2610271439950769E11,
            "count": 130.0
          },
          "1470417780000": {
            "avg": 1.26102137856E11,
            "count": 4.0
          },
          "1470417840000": {
            "avg": 1.26101682176E11,
            "count": 4.0
          },
          "1470417900000": {
            "avg": 1.26101118976E11,
            "count": 4.0
          }
        },
        "numBins": 60,
        "binDurationMillis": 60000,
        "aggregationType": [
          "AVG"
        ]
      }
    },
    "cpuUsagePercentPerHour": {
      "name": "cpuUsagePercentPerHour",
      "latestValue": 0.01548839632318137,
      "accumulatedValue": 116.30780660032818,
      "version": 141,
      "lastUpdateMicrosUtc": 1470417944799006,
      "kind": "com:vmware:xenon:common:ServiceStats:ServiceStat",
      "timeSeriesStats": {
        "bins": {
          "1470417720000": {
            "avg": 0.8984681879328744,
            "count": 129.0
          },
          "1470417780000": {
            "avg": 0.0586236815181815,
            "count": 4.0
          },
          "1470417840000": {
            "avg": 0.025864196283287663,
            "count": 4.0
          },
          "1470417900000": {
            "avg": 0.016864711445377857,
            "count": 4.0
          }
        },
        "numBins": 60,
        "binDurationMillis": 60000,
        "aggregationType": [
          "AVG"
        ]
      }
    },

.....
.....
}
```

# Backup/Restore

see [[Backup-Restore]]