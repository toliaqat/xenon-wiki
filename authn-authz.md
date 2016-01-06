This document has been deprecated and will be removed shortly. Please see:

[Authentication and Authorization Design](Authentication-and-Authorization-Design)



# Authentication & authorization

> **This is a work in progress**

## Entity modeling

* Authentication is done for _users_.
* Authorization is done through _roles_.
* _Roles_ belong to _users_ through _user groups_.
* _Roles_ apply to _resources_ through _resource groups_.
* _Roles_ allow/deny HTTP verbs to be executed by _users_ against _resources_.
* A _user group_ is expressed as a query over users.
* A _resource group_ is expressed as a query over resources.

### Users

User identity is captured by email address. The identity does not need to maintained in Xenon; an external identity provider can be used. However, the user must have at least a single document that represents it, so that it can be returned as part of a user group query. The kind of document does not matter, as long the backing document has an "email" field equal to the user's email address.

For example, you can have users represented by their credentials alone (through the `AuthCredentialService`, which has an "email" field), by a special `LDAPUserService` which keeps in sync with an upstream LDAP service, or by something else. In the case an external identity provider is used, a document representing the user can be created on demand.

### User group

User groups are expressed as queries over users. This means it is possible to have groups for "users named Jane" or "users who were active in the last week". As long as a property (native or mirrored from an upstream identity provider) is modeled in the user service, it can be queried for, and used to create a group. This also means it is possible to mirror external groups, as long as that external group membership is modeled as property of the user state (such that it can be queried for).

* UserGroupService -- `/core/authz/user-groups`

### Resource groups

Similar to user groups. Resource groups are expressed as queries over resources. In the initial implementation of authz in Xenon, it is only possible to specify resources by their URI path prefix (allowing or denying access to an path hierarchy).

* ResourceGroupService -- `/core/authz/resource-groups`

### Roles

Roles bind user groups to resource groups. Roles have the following attributes:

* user group link
* resource group link
* verbs (set of HTTP verbs the user can use with this role)

One or more roles apply to a specific user if that user is contained in the user groups specified by those roles.

* RoleService -- `/core/authz/roles`

## Service interaction

### Authentication

Multiple authentication backends can be in use at the same time. One set of users may be using a keypair, while another set uses a hashed password which is checked against an LDAP backend, while yet another uses OAuth2 with some external identity provider. Every one of these authentication services has their own interaction pattern. For example: in order to use the keypair, the server asks poses the user with a challenge, or in order to use OAuth2, the user might be redirected to an external URL. Whatever the interaction, the net result is that the user is given a token it can use in future interaction with Xenon. This token is passed as a cookie, which means that any HTTP client that has support for a cookie jar is supported out of the box.

### Authorization

Every subsequent request after authenticating will carry the token and will be subject to authorization. A request is identified by the user who made the request, the HTTP verb, and the URI path for the request. This 3-tuple is used as input to the authorization service, which will return a simple true or false, to either allow or deny the request to be handled. The authorization service is run on every Xenon instance and talks to the local index. User and role information is expected to be available on all nodes in the default node group. The authorization service retrieves the state associated with the requested URI and tests it against the query specification in every applicable role.

This means the runtime cost of the authorization service is equal to lookup of the state for the requested service and N resource group checks against the applicable roles (before applying any optimization over the query specification that is the composite of the queries in the roles, which is equivalent to the resource group query specification in all roles in disjunctive normal form).

The authorization service performs caching where possible to amortize any query cost.

## Tenancy

It is possible to implement tenancy on top of the entities and interactions described in this document. Every tenant will have its own set of roles, that allow users who belong to that tenant (through the user group specific to that tenant) access to resources that belong to that same tenant. This can be expressed by adding query terms to the queries for the user group and resource group.

## Notes

* Forwarding logic is to be executed **before** authorization logic (resources in the resource group may not be present on the entry node)

