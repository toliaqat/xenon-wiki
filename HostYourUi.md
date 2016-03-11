# Overview
Xenon provides built-in support for interacting with your services through user interface from your browser. You have the option to load and serve your factory and instance services through the default UI (the resources reside on the disk). To access the default UI point your browser at localhost:8000/selflink-to-service/ui. A different option is define your own custom UI by using the HTML_USER_INTERFACE_OPTION on the service that you want to serve with the custom resources. 

The UI support is flexible enough to allow you to point any number or services to the default or the custom UI by sharing the custom endpoint.

This section describes how:
1. [access the default Xenon UI]./(HostYourUi#access-the-default-ui)
2. [host the same custom UI for multiple services](./HostYourUi#host-custom-ui-for-a-collection-of-micro-services)

## Access the default UI
Xenon provides a generic default user interface, and which will serve any service defined. The default UI is built with AngularJS and is served as a separate service (UiService). The main entry point to the default UI is http://localhost:8000/core/ui/default which loads a basic dashboard (it will gradually include more information as Xenon matures.) This interface allows for drilling down to more details in the system (system information details on the graphs, links to available services etc.)
![dashboard](./dashboard.png)


To access the default UI point your browser to localhost:8000/service-self-link/ui (for ExampleService point to http://localhost:8000/core/examples/ui) and get an overview of the factory along with basic functionality for querying for services, adding/editing/deleting instances, and preview stats or service instances, when available.)

![ServiceLandingPage](./ServiceLandingPage.png)

## Host custom UI 
**Note**: You can define the custom UI of your service using different front end frameworks (JavaScript, jQuery, etc.) In practice, we recommend the use of AngularJS, since it provides valuable features (such as routing) and allows to write clean UI with few lines of code.

### Host custom UI for a single microservice
**Note**: Find the complete code for this tutorial in the xenon-samples of the module of xenon source.  

To define custom UI for a service, follow the steps below:

**Step 1**: Create a new service which will be responsible for serving the custom UI. An example would be the SampleServiceWithSharedCustomUi and it would look like the code below:

```
/*
 * Copyright (c) 2015 VMware, Inc. All Rights Reserved.
 */

package com.vmware.xenon.services.samples;

import com.vmware.xenon.common.Operation;
import com.vmware.xenon.common.StatelessService;
import com.vmware.xenon.common.UriUtils;
import com.vmware.xenon.common.Utils;
import com.vmware.xenon.services.common.ServiceUriPaths;

public class SampleServiceWithSharedCustomUi extends StatelessService {
    public static final String SELF_LINK = ServiceUriPaths.CUSTOM_UI_BASE_URL;

    public SampleServiceWithSharedCustomUi() {
        super();
        toggleOption(ServiceOption.HTML_USER_INTERFACE, true);
    }

    @Override
    public void handleGet(Operation get) {
        String serviceUiResourcePath = Utils.buildUiResourceUriPrefixPath(this);
        serviceUiResourcePath += "/" + ServiceUriPaths.UI_RESOURCE_DEFAULT_FILE;

        Operation operation = get.clone();
        operation.setUri(UriUtils.buildUri(getHost(), serviceUiResourcePath))
                .setReferer(get.getReferer())
                .setCompletion((o, e) -> {
                    get.setBody(o.getBodyRaw())
                            .setContentType(o.getContentType())
                            .complete();
                });

        getHost().sendRequest(operation);
        return;
    }
}
``` 

**Step 2**: Once SampleServiceWithSharedCustomUi was created it needs to be started in the host:
```
super.startService(
                Operation.createPost(UriUtils.buildUri(this, SampleServiceWithSharedCustomUi
                        .class)), new SampleServiceWithSharedCustomUi());
```

**Step 3**: Next, the actual UI resources need to be added. Place the UI resources under resources/ui/service-package-name. For the SampleServiceWithSharedCustomUi the UI files are under resources/ui/com/vmware/xenon/services/samples/SampleServiceWithSharedCustomUi. 
The custom UI for SampleServiceWithSharedCustomUi is built in AngularJS, and a typical folder structure for that would be the following:

![angularApp-structure](./angularApp-structure.png)

**Step 4**: Next, for the service that will use above custom UI set the **ServiceOption.HTML_USER_INTERFACE to true** inside the constructor. When this option is set to true the back end expects to find the UI resources related to this service under resources/ui/com/vmware/xenon/services/samples/SampleServiceWithCustomUi (next step). 
```
public SampleServiceWithCustomUi() {
        super(SampleServiceWithCustomUiState.class);
        super.toggleOption(ServiceOption.PERSISTENCE, true);
        super.toggleOption(ServiceOption.REPLICATION, true);
        super.toggleOption(ServiceOption.INSTRUMENTATION, true);
        super.toggleOption(ServiceOption.OWNER_SELECTION, true);
        super.toggleOption(ServiceOption.HTML_USER_INTERFACE, true);
    }
```

**Step 5**: Every time you point your browser to services-using-custom-ui-self-link/ui the host looks for an index.html file. That index.html needs to be placed under resource/ui/your-service-package. Form the SampleServiceWithCustomUi it needs to be under resources/ui/com/vmware/xenon/services/samples/SampleServiceWithCustomUi and it looks like the below:
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta content="utf-8" http-equiv="encoding">
</head>

<body>
<script>
    var pathname = window.location.pathname;
    pathname = pathname.substring(0, pathname.length - 3);
    window.location.href = window.location.origin + "/core/ui/custom#" + pathname +
    "/home";
</script>
</body>
</html>
```

The index of SampleServiceWithCustomUi will then get extract the self-link of the service from the window location and redirect to the custom UI. The URL for the custom UI would look like: **http://localhost:8000/core/ui/custom#/core/customUiExamples/home** (for factory) or **http://localhost:8000/core/ui/custom#/core/customUiExamples/3668a6c6-fa0f-49fc-8ba7-e3b1e5133a1f/home** (for an instance). In this case, the basic custom UI will look like this: 

![custom_ui](./custom_ui.png)


**Step 6**: Since the SampleServiceWithCustomUi has the HTML_USER_INTERFACE option on, the corresponding factory service (SampleFactoryServiceWithCustomUi) will get that option as well. As expected, a similar index.html file needs to be in place for the factory service. Under resources/ui/com/vmware/xenon/services/samples/SampleFactoryServiceWithCustomUi add an identical index.html file as in the previous step (which will again redirect to the same custom UI). 


### Routing
When creating a custom user interface you would need to use different end points to serve different pages. To achieve this we are taking advantage of the AngularJs routing capabilities. In the config of the angular app that will serve the UI you can define set the different paths that you can serve using $routeProvider. The example configuration of the custom shared UI looks like:
```
/*
 * Copyright (c) 2015 VMware, Inc. All Rights Reserved.
 */

'use strict';
/* App Module */
var customUiApp = angular.module('customUiApp', ['ngRoute', 'ngResource', 'nvd3', 'json-tree']);

customUiApp .config(['$routeProvider', function ($routeProvider) {
    $routeProvider.
        when("/", {
            templateUrl: CONSTANTS.UI_RESOURCES + 'pages/home/homeView.html'
        }).
        when("/:path/:serviceName/home", {
            templateUrl: CONSTANTS.UI_RESOURCES + 'pages/home/homeView.html'
        }).
        when("/:path/:serviceName/:instanceId/home", {
            templateUrl: CONSTANTS.UI_RESOURCES + 'pages/home/homeView.html'
        }).
        when("/:path/:serviceName/addService", {
            templateUrl: CONSTANTS.UI_RESOURCES + 'pages/addPage/addService.html'
        }).
        otherwise({
            redirectTo: '/'
        });

}]);
``` 
In more detail, if you pointed your browser to http://localhost:8000/core/ui/custom# you would be redirected to the home.html (in the first when clause above). If you wanted to add an additional service you can add a link to your html views that will take you there. The link would look like http://localhost:8000/core/ui/custom#/core/customUiExamples/addService (the fourth when clause in the above config). In similar fashion, you can extend as needed.
