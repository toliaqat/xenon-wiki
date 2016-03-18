# Overview
Starting with 0.7.5 a ServiceHost can describe all deployed services using Swagger 2.0. The descriptor itself is served by a stateless service. This service is started as any other on a host:
```java
SwaggerDescriptorService swagger = new SwaggerDescriptorService();
// provide general API info
Info info = new Info();
info.setDescription("description");
info.setTermsOfService("terms of service etc.");
info.setTitle("title");
info.setVersion("1.0");
swagger.setInfo(info);

// option to exclude some services
swagger.setExcludedPrefixes("/core/authz/");

host.startService(swagger);
```

# REST API

```/discovery/swagger```

## GET
The descriptor is available at /discovery/swagger. It can be retrieved in JSON or YAML format depending on the "Accept" header.

# SwaggerUI
SwaggerUI is deployed as custom UI of the stateless service. It is available at /discovery/swagger/ui.
![swagger-ui](./images/swagger-ui.png)

# Limitations
* stateless service are not supported currently
* swagger annotations are ignored
* Authentication is not supported