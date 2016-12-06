# 1 Overview
Authentication and authorization is built into Xenon and controls access to services.

When authentication and authorization are enabled, all request to a stateful Xenon service will be subject to two additional checks:

 1. Is the request on behalf of a valid user?
 2. Is that user authorized to perform the desired action the service?

Any unauthorized access will result in a 403 (Forbidden) response from Xenon.

Note that we do not have any authorization checks on stateless services at this time (it is in the works). Stateless services typically interact with other stateful services and that access is subject to authorization checks.

Also, FactoryService instances are stateless services which can be invoked by any user. POSTs to factories result in a stateful service instance being created and that operation will complete successfully only if the user who invoked the factory service has the right privileges on the resulting stateful service. Any user can query a factory service, but the the response will only include services they are authorized to access.

# 2 Authentication and Authorization Concepts

## 2.1 Entity modeling

* Authentication is done for _users_.
* Authorization is done through _roles_.
* _Roles_ belong to _users_ through _user groups_.
* _Roles_ apply to _resources_ through _resource groups_.
* _Roles_ allow/deny HTTP verbs to be executed by _users_ against _resources_.
* A _user group_ is expressed as a query over users.
* A _resource group_ is expressed as a query over resources.

## 2.2 Users

User identity is captured by email address. The identity does not need to maintained in Xenon; an external identity provider can be used. However, the user must have at least a single document that represents it, so that it can be returned as part of a user group query. The kind of document does not matter, as long the backing document has an "email" field equal to the user's email address.

For example, you can have users represented by their credentials alone (through the `AuthCredentialService`, which has an "email" field), by a special `LDAPUserService` which keeps in sync with an upstream LDAP service, or by something else. In the case an external identity provider is used, a document representing the user can be created on demand.

* UserService -- `/core/authz/users`

## 2.3 User group

User groups are expressed as queries over users. This means it is possible to have groups for "users named Jane" or "users who were active in the last week". As long as a property (native or mirrored from an upstream identity provider) is modeled in the user service, it can be queried for, and used to create a group. This also means it is possible to mirror external groups, as long as that external group membership is modeled as property of the user state (such that it can be queried for).

* UserGroupService -- `/core/authz/user-groups`

## 2.4 Resource groups

Similar to user groups. Resource groups are expressed as queries over resources. In the initial implementation of authz in Xenon, it is only possible to specify resources by their URI path prefix (allowing or denying access to an path hierarchy).

* ResourceGroupService -- `/core/authz/resource-groups`

## 2.5 Roles

Roles bind user groups to resource groups. Roles have the following attributes:

* User group link
* Resource group link
* Policy: set of HTTP verbs the user can use with this role

One or more roles apply to a specific user if that user is contained in the user groups specified by those roles.

* RoleService -- `/core/authz/roles`

# 3 Service interaction

## 3.1 Authentication

Multiple authentication backends can be in use at the same time. One set of users may be using a keypair, while another set uses a hashed password which is checked against an LDAP backend, while yet another uses OAuth2 with some external identity provider. Every one of these authentication services has their own interaction pattern. For example: in order to use the keypair, the server asks poses the user with a challenge, or in order to use OAuth2, the user might be redirected to an external URL. Whatever the interaction, the net result is that the user is given a token it can use in future interaction with Xenon. This token is passed as a cookie, which means that any HTTP client that has support for a cookie jar is supported out of the box.

## 3.2 Authorization

Every subsequent request after authenticating will carry the token and will be subject to authorization. A request is identified by the user who made the request, the HTTP verb, and the URI path for the request. This 3-tuple is used as input to the authorization service, which will return a simple true or false, to either allow or deny the request to be handled. The authorization service is run on every Xenon instance and talks to the local index. User and role information is expected to be available on all nodes in the default node group. The authorization service retrieves the state associated with the requested URI and tests it against the query specification in every applicable role.

This means the runtime cost of the authorization service is equal to lookup of the state for the requested service and N resource group checks against the applicable roles (before applying any optimization over the query specification that is the composite of the queries in the roles, which is equivalent to the resource group query specification in all roles in disjunctive normal form).

The authorization service performs caching where possible to amortize any query cost.

Note that if a user is authorized to access a service, they are also authorized to access the service's helpers, like /stats.

## 3.3 Tenancy

It is possible to implement tenancy on top of the entities and interactions described in this document. Every tenant will have its own set of roles, that allow users who belong to that tenant (through the user group specific to that tenant) access to resources that belong to that same tenant. This can be expressed by adding query terms to the queries for the user group and resource group.

# 4 Privileged Services
Privileged services are services that can execute operations in a 'system context'. When running in the system context we bypass the regular security checks and allow access to all services. Think of running in the system context as running as root on unix and a privileged service as a process with the setuid bit set.

In order to control what services can run as privileged, it must be explicitly marked as privileged before it is started by the host. For example if we need to start the ‘UserInitializationService’ as privileged, we would do the following in ServiceHost:

```java
host.addPrivilegedService(UserInitializationService.class);
host.startService(Operation.createPost(UriUtils.buildUri(host, UserInitializationService.class)),
                  new UserInitializationService());
```

`addPrivilegedService` is a protected method and hence can be invoked only by the ServiceHost instance (or its subclasses).

# 5 Authorization token cache

Every xenon service host maintains a cache of a logged in user’s authorization context instance keyed off the auth token. This helps associate an incoming user request with the right security context very quickly. The authorization context is created as part of request dispatch flow in cases when the cache does not have an entry for the token associated with an incoming request. All subsequent requests on behalf of the user reuse the authorization context. 

The authorization context obviously becomes invalid whenever any property of the service representing the user, UserGroupService, RoleService or ResourceGroupService changes. When that happens, all auth token cache instances that are potentially impacted by the change are invalidated.

The logic to clear the cache must kick in after the change to the service has been replicated and the document index updated on all nodes. This is achieved by triggering the cache clear logic via the processCompletionStageUpdateAuthzArtifacts() method. This method is invoked by the framework for any stateful service after the update has been replicated to all nodes and the index updated. Additionally, any update to auth artifacts must be done with the replication quorum head set to “all” to ensure the updates are propagated to all nodes irrespective of what the quorum is set to.

All authz services override processCompletionStageUpdateAuthzArtifacts() and are responsible for identifying the users who are impacted by the change in the service. This typically means traversing the relationship between the authz service back to the user and triggering the code to clear the auth token cache. For example, if there was a change in a ResourceGroupService instance, all RoleService instances that have a reference to the ResourceGroupService identified, all UserGroupService instances that are referenced by the RoleService instances followed, and finally all user service instances that are referenced by the UserGroupService instances identified to build out the target set of authorization context cache entries that needs to be cleared. This logic runs on every node in a node group

Helper methods in AuthorizationCacheUtils helps implement this logic for the various authz service types.

Once the target set of user service instances affected by an authz service update has been identified, the actual task of clearing out the cache lies with AuthorizationTokenCacheService. This service is a stateful singleton service that is in-memory, non-replicated and started up as a core service along with the authorization service as part of service host start. This service can be PATCHed with the payload being the user service link for which the cache needs to be invalidated. The patch handler in turn calls out to AuthorizationContextService with the “xn-clear-auth-cache” pragma set so that the service host cache can be cleared.  AuthorizationTokenCacheService delegates the actual cache clear to AuthorizationContextService so that the cache clear operation can be coordinated with any in-flight operations to populate the cache to ensure we do not end up with stale auth token cache entries - if there is a request to populate the cache for a user and a cache clear request comes in for that user, the request in flight is aborted and the process of creating a cache entry for the user restarted.

This sequence of events can be represented by the following sequence diagram:

[[images/developer-guide/auth-token.png]]

Any service (within the same node group or externally) that is interested in being updated on an auth token cache clear event can register to be notified on an update to the singleton AuthorizationTokenCacheService instance. For example, if external auth is being used by a xenon node group and it wants to ensure its local auth token cache is updated based on any changes to the remote auth xenon node group, it can register for notifications from AuthorizationTokenCacheService on the auth provider and act as a relay clearing out the its own auth token cache by invoking a PATCH on the local AuthorizationTokenCacheService instance.


# 6 Notes

* Forwarding logic is to be executed **before** authorization logic (resources in the resource group may not be present on the entry node)

* Authentication and authorization are turned off by default at this time and can be turned on by passing in the flag —-isAuthorizationEnabled=true to the ServiceHost (or it's subclasses) during startup. If you subclass ServiceHost, you can enable it with the ServiceHost.setAuthorizationEnabled method.

* The [ExampleServiceHost](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/ExampleServiceHost.java) has an example of creating users so you can use them "out of the box"

* Today, only the BasicAuthorizationService has been implemented. This can authenticate local users using password. In the future, other authentication services will be implemented, including services that can interact with external authentication systems such as ActiveDiredtory.

# 7 More Information

The [./Authentication-and-Authorization-Tutorial](Authentication and Authorization Tutorial) has examples that walk you through using authentication and service from the command-line as well as examples of creating users, user groups, resource groups and roles in Java.

This [presentation] (https://github.com/vmware/xenon/blob/master/contrib/docs/xenon-authz.pptx) gives an example of authz in action
