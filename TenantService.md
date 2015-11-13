# Overview

The tenant service exposes a REST API to manage tenants in Xenon. Tenants can also be nested by specifying a parentLink so that a "super" tenant can manage multiple tenants.

## Authorization

Service resources can refer to a specific tenant using a **tenantLink** field in the resource PODO, for example:
```
public static class ServiceFooState extends ServiceDocument {
    public String tenantLink;
}
```
The authorization service can then filter out resources with a tenant role using a query on the tenantLink field. See [this page](./authn-authz) for more details on authn/authz.

## REST API

### URI Path
```
/core/tenants
```

### POST
TenantState
```json
{
   "id": <id>
   "name": <name>
}
```

Creates a new tenant with the specified ID and name. If ID is not set, the system automatically generates a random UUID. 

### GET
Retrieves tenant information.

### PATCH
TenantState
```json
{
   "name": <name>
}
```

Modifies an existing tenant. Tenant ID cannot be modified.
