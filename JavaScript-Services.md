## Overview

WebUI front-end applications are often supposed to receive PUSH notifications from backend. In DCP this problem is solved via ability to register remote DCP services attached through WebSocket connection to a node. Such services may be invoked by any other services and they may be used as subscribers for other services updates as well.

## WebSocket Connection

DCP host provides common pure JavaScript library for establishing and managing WebSocket connections to a DCP node. This library is available at:

```
/user-interface/resources/com/vmware/dcp/common/WebSocketService/ws-service-lib.js
```

The above library defines a `WSL` object which defines `WebSocketConnection` class. An instance of this class may be created using the following constructor:

```js
var connection = new WSL.WebSocketConnection("/core/ws-endpoint")
```

## In-Browser JavaScript DCP services

Since In-Browser JavaScript DCP services cannot have regular public URIs, an auto-generated ephemeral URIs are used to identify such services. These URIs are assigned on request by a DCP node. These URIs are only valid until WebSocket connection is alive and service is not stopped. In order to create a new JavaScript DCP service, it is necessary to provide `handleRequest(op)` method which encapsulates service implementation and may handle any type of REST request.

Service creation involves a remote call to DCP node for an ephemeral service URI, so service creation result is only available as a callback. Here is an example of a service creation:

```js
connection.createService(
    function (op) {
        try {
            // ### Handle operation here ###
        } finally {
            op.complete();
        }
    },
    function (wss) {
        // ### Handle created WebSocketService here ###
    });
```

## Operation object

`Operation` object has `id`, `action`, `statusCode` and `body` fields which correspond to the same fields of `com.vmware.dcp.common.Operation` class. One of methods `complete()`, `fail(exception)` or `fail(exception, body)` should be used to complete received operation.

## WebSocketService object

`WebSocketService` object has `uri` field which is set to ephemeral service URI accessible from within DCP cluster. It also has `subscribe`, `unsibscribe` and `stop` methods. `stop` methods un-registers the servers. Other two methods are described below.

## JavaScript observers

It is possible to use `wss.uri` explicitly to subscribe JavaScript service on updates for any other observable DCP service. However in case when used closes browser page, observed service cannot know that observer has gone forever and subscription record gets stuck indefinitely in subscribers list.

`WebSocketService` object `subscribe(uri)` and `unsubscribe(uri)` methods can be used to register and unregister the service as an observer for other service. `uri` argument is a URI for subscriptions sub-service. Subscriptions registered with `subscribe(uri)` method will be automatically removed when WebSocket connection is closed or service is stopped.

## Example

dcp-host project carries an example JavaScript service which subscribes to examples service factory updates. Demo page is available at: http://localhost:8000/user-interface/resources/com/vmware/dcp/ui/UiService/features/ws/ws-demo.html

Links to all added example service documents are displayed whenever new example documents are created.

Minimalistic JavaScript DCP service example is provided below:

```html
<html>
<head>
    <title>WebSocket Service Demo</title>
    <script type="text/javascript"
            src="/user-interface/resources/com/vmware/dcp/ui/UiService/libs/jQuery/jquery-2.1.0.js"></script>
    <script type="text/javascript"
            src="/user-interface/resources/com/vmware/dcp/common/WebSocketService/ws-service-lib.js"></script>
    <script type="text/javascript"
            src="/user-interface/resources/com/vmware/dcp/ui/UiService/features/ws/ws-demo.js"></script>
</head>
<body>
    <h1>Created example objects</h1>
    <div id="objects-created"></div>
</body>
</html>
```

```js
var wsEndpoint = "/core/ws-endpoint";
var exampleServiceSubscriptions = "/core/examples/subscriptions";

new WSL.WebSocketConnection(wsEndpoint).createService(
    function (op) {
        try {
            $("#objects-created").append('<a href="' + op.body.documentSelfLink + '">' + op.body.name +
                ' (' + op.body.documentSelfLink + ')</a><br/>');
        } finally {
            op.complete();
        }
    },
    function (wss) {
        wss.subscribe(exampleServiceSubscriptions);
    }
);
```
## WebSocket communication protocol

WebSocket communication protocol is describe in `com.vmware.dcp.common.http.netty.NettyHttpClientRequestHandler` JavaDoc.