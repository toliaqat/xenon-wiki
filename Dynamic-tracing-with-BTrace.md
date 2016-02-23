Sometimes logging and operation tracing give too much or too little information for tracing a specific issue.

BTrace is a tool for dynamically tracing and instrumenting java code (the java equivalent of DTrace).
You write a probe in java then dynamically insert it into a running Xenon host.

`
$ PID=$(jps | awk '/my-service-host-1.0.0.jar/ { print $1 }')
$ bin/btrace -cp /path/to/my-service-host-1.0.0.jar $PID Xenon.java 
`

Some examples of things that can be done for Xenon with BTrace:

1. Print all the the operations being sent in the host (operation details such as URI, body as well as the stacktrace leading to the operation)

`

package com.vmware.xenon.btrace;

import com.sun.btrace.annotations.*;
import static com.sun.btrace.BTraceUtils.*;
import java.net.*;

@BTrace public class Xenon {
    @OnMethod(
        clazz="+com.vmware.xenon.common.ServiceClient",
        method="send")
    public static void onClientSend(com.vmware.xenon.common.Operation operation) {
        // print the toString of the Operation (alternatively can print only specific fields):
        println("Client Send: " + operation);

        // optionally print the stacktrace useful for understanding which service the operation originated from
        jstack(); 
    }
}

`

This is similar to the operation tracing feature, but you can filter for specific URIs, services or include more relevant information.


2. Create a histogram of accessed URIs:

`

package com.vmware.xenon.btrace;

import com.sun.btrace.annotations.*;
import static com.sun.btrace.BTraceUtils.*;
import java.net.*;
import java.util.Map;
import java.util.concurrent.atomic.AtomicInteger;

@BTrace public class Xenon {
    public static String ignorePath = "/core/node-groups/default";
    public static String ignoreReferer = "/core/document-index";
    private static Map<String, AtomicInteger> histo = Collections.newHashMap();

    @OnMethod(
        clazz="+com.vmware.xenon.common.ServiceClient",
        method="send")
    public static void onClientSend(com.vmware.xenon.common.Operation operation) {
        Object refererUri = Reflective.get(Reflective.field("com.vmware.xenon.common.Operation", "referer"), operation);
        String referer = str(Reflective.get(Reflective.field("java.net.URI", "path"), refererUri));
        if (compare(ignoreReferer, referer)) {
            return;
        }
        Object uri = Reflective.get(Reflective.field("com.vmware.xenon.common.Operation", "uri"), operation);
        String path = str(Reflective.get(Reflective.field("java.net.URI", "path"), uri));
        if (compare(ignorePath, path)) {
            return;
        }

        AtomicInteger ai = Collections.get(histo, path);
        if (ai == null) {
            ai = Atomic.newAtomicInteger(1);
            Collections.put(histo, path, ai);
        } else {
            Atomic.incrementAndGet(ai);
        } 
    }

    @OnTimer(4000) 
    public static void print() {
        if (Collections.size(histo) != 0) {
            printNumberMap("Path Histogram", histo);
        }
    }
}

`
