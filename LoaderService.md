## Overview
The LoaderService is an extension to DCP native service hosting model, allowing to dynamically add new services to a running DCP host without restarting. Dynamic service loading is implemented using a custom service host which starts the LoaderService factory and the default instance. The default instance loads services from a predefined "services" directory under the host storage directory, using UrlClassLoader and reflection.
The model allows creation of new loader service instances to load services from other locations, including file system, online repository (TBD), etc. 

## Project 
The dcp-loader project source code is located at: https://github.com/vmware/dcp/tree/master/dcp-loader

## Loader Service Host
Custom Service Host is a simple extension of a service host that starts the LoaderService factory service as well as creates the default instance of the LoaderService service
```
    @Override
    public ServiceHost start() throws Throwable {
		...
        // start the loader service factory
        super.startService(
                Operation.createPost(UriUtils.buildUri(this, LoaderFactoryService.class)),
                new LoaderFactoryService());

        // Start the default instance.
        // Setting target replicated to ensure the factor is loaded first
        Operation post = LoaderFactoryService
                .createDefaultPostOp(this)
                .setTargetReplicated(true)
                .setReferer(UriUtils.buildUri(this, ""));
        sendRequest(post);
        return this;
    }
```

## LoaderService
When the factory is started the default instance is created, that handles loading of services from a predefined file system location.
Posting GET to the factory produces the following:

## GET /core/loader
```json
{
  "documentLinks": [
    "/core/loader/default"
  ],
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "b9384176-c061-493c-a49a-94009044e287"
}
```

The default instance loads services during the maintenance cycle (10 sec currently) and checks if there are changes compared to the previous update. If changes are present, factory and singleton classes will be loaded and started. The list of loaded service packages and according factory and singleton service classes is stored as part of the service document:

## GET /core/loader/default
```json
{
  "loaderType": "FILESYSTEM",
  "path": "services",
  "servicePackages": {
    "file:/tmp/dcp/8000/services/dcp-example-services-0.0.2-SNAPSHOT.jar": {
      "name": "dcp-example-services-0.0.2-SNAPSHOT.jar",
      "fileUpdateTimeMillis": 1431125710000,
      "serviceClasses": {
        "com.vmware.dcp.examples.ExampleSingletonService": "/loader/examples/bar",
        "com.vmware.dcp.examples.ExampleFooFactoryService": "/loader/examples/foo"
      }
    }
  },
  "documentVersion": 1,
  "documentKind": "com:vmware:dcp:common:LoaderService:LoaderServiceState",
  "documentSelfLink": "/core/loader/default",
  "documentUpdateTimeMicros": 1431125761716000,
  "documentExpirationTimeMicros": 0
}
```

## Service Packages
To avoid introducing extra dependencies, the dynamic service loader requires factory and singleton classes within the service package to follow these rules:
* Dynamically loaded service classes need to implement com.vmware.dcp.common.Service interface
* Dynamically loaded service classes need to have name ending with either "Factory" or "Service", e.g. "ExampleFactory" or "BarService".
* Dynamically loaded service classes  need to have a public static String SELF_LINK field, for example:
```
public static final String SELF_LINK = UriUtils.buildUriPath("loader","examples","foo");
```

The loader service first identifies dynamically loaded classes by name ending with "Factory" or "Service". Matching classes will be loaded using a URL class loader, and then checked for assignability to com.vmware.dcp.common.Service.
Here is a code snippet from a sample factory service implementation used for dynamic loading. 

```
public class ExampleFactoryService extends FactoryService {
	public static final String SELF_LINK = UriUtils.buildUriPath("loader","examples","foo");
	public ExampleFactoryService() {
		super(CredentialKindServiceState.class);
	}
	@Override
	public Service createServiceInstance() throws Throwable {
		return new CredentialKindService();
	}
}
```

## Setup Instructions:
* setup and run the LoaderServiceHost from dcp-loader project. The LoaderService is not yet part of the default ServiceHost, so in order to enable dynamic loading you need to run the LoaderServiceHost extension.
* create a directory called "services" inside the host storage directory (e.g. $TMPDIR/dcp/8000 or /tmp/dcp/8000) 
* drop the jar file containing service classes into the "services" directory
* give it 10 sec and send a get to the default loader instance: GET /core/loader/default. If everything is ok it should list your service in the list of service packages and service classes found in them.
* start sending requests to your services
