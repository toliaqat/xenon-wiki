# 1.0 Overview
This page gives you an introduction to authentication and authorization in Xenon. 

This tutorial assumes that you have gone through the introductory [Example Service Tutorial](Example-Service-Tutorial) as well as the [Introduction to Service Queries](./Introduction-to-Service-Queries). Those tutorials did not enable authentication and authorization. In a production environment, you'll certainly enable them. This tutorial will help you understand how it works. 

This tutorial assumes you have installed Xenon and can run it from the command-line. It also assumes you have installed the "curl" command-line tool. If you don't want to use curl, the examples should be easily converted to use another tool, such as Postman. It also assumes you have installed the [jq tool](https://stedolan.github.io/jq/), which we use for formatting JSON responses from Xenon. If you do not have it installed, you can simply remove it from the commands below. 

If you are interested, you can also read the [./Authentication-and-Authorization-Design](Authentication and Authorization Design)

# 2.0 Starting the Xenon host

We are going to start the Xenon host differently than the previous tutorials:

1. We are going to start a different host, the ExampleServiceHost. This host provides extra arguments to make it easier to work with authorization.
2. We are going to provide arguments to enable authentication and authorization as well as create two users. The two users will be:
  * User "root@localhost" with password "changeme"
  * User "example@localhost" with password "changeme"

Users in Xenon are always identified by an email address. 

```
% java -cp xenon-host/target/xenon-host-0.4.0-SNAPSHOT-j-with-dependencies.jar com.vmware.xenon.services.common.ExampleServiceHost --sandbox=/tmp/xenon --isAuthorizationEnabled=true --adminUser=admin@localhost --adminUserPassword=changeme --exampleUser=example@localhost --exampleUserPassword=changeme

[0][I][1452037260285][ExampleServiceHost:8000][startImpl][ServiceHost/cd70e1c8 listening on 127.0.0.1:8000]
[1][I][1452037260557][ExampleServiceHost:8000][printUserDetails][Created user example@localhost (/core/authz/users/13e016c6-d69e-41f0-9ffc-e7e9939227a0) with credentials, user group (/core/authz/user-groups/ef9c5dea-ced1-4375-9ce3-6b5c0216ac12) resource group (/core/authz/resource-groups/d7eea0b5-d814-4d63-8e29-68b32dbb6995) and role(/core/authz/roles/7ce7c6d7-6d40-41da-842c-6af40fd67d32)]
[2][I][1452037260557][ExampleServiceHost:8000][printUserDetails][Created user admin@localhost (/core/authz/users/a996822b-b2df-4a3d-a96b-f560cdf9f134) with credentials, user group (/core/authz/user-groups/86b928df-261a-48ff-9d56-eff01c4e75eb) resource group (/core/authz/resource-groups/406c1eae-54ad-4313-b0ff-a7a3075c905a) and role(/core/authz/roles/db12ffe9-9245-4314-98b0-93de5fd6d068)]
```

The output is a bit long, but you can learn quite a bit from it. For example, we have created a user named admin@localhost:
* Each user is represented by a user service. The admin user is at: /core/authz/users/a996822b-b2df-4a3d-a96b-f560cdf9f134
* We made a user group that contains just this user. The admin's user group is at: /core/authz/user-groups/86b928df-261a-48ff-9d56-eff01c4e75eb
* We made a resource group from the admin user that contains a set of resources: /core/authz/resource-groups/406c1eae-54ad-4313-b0ff-a7a3075c905a
* We made a role that connects together the admin's user and resource groups: /core/authz/roles/db12ffe9-9245-4314-98b0-93de5fd6d068

We'll look at this in detail below. 

# 3.0 Exploring authz from the command-line

## 3.1 An unauthorized user
An unauthorized user can query (GET) on any resource, but they'll get back an empty document. For example:

```
% curl http://localhost:8000/core/authz/users -H "Content-Type: application/json" 
{
  "documentLinks": [],
  "documentCount": 0,
  "queryTimeMicros": 998,
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "6682ac24-fac5-4ff3-b9ae-66db2eb3fa3f"
}
```

If an unauthorized user tries to modify data, they are forbidden: (output trimmed for simplicity)
```
% curl -s -X POST -d '{"email":"root@localhost"}' http://localhost:8000/core/authz/users -H "Content-Type: application/json" | jq .
{
  "message": "forbidden",
  "stackTrace": [
    ... output trimmed ...
  ],
  "statusCode": 403,
  "documentKind": "com:vmware:xenon:common:ServiceErrorResponse"
}
```

## 3.2 Authentication: Getting an auth token
All API operations require an auth token. The API workflow is:

* POST user credentials to an authentication service. Currently this is /core/authn/basic, which authenticates local users based on password, but other authentication services (e.g. ActiveDirectory) will be added in the future. 
* If the credentials are correct, the user will receive an _auth token_
* Future API calls should provide the auth token as a header named "x-xenon-auth-token"

Alternatively, clients can use cookies: the auth token is embedded in the `xenon-auth-cookie`. The auth token header is the preferred approach, but both are acceptable. 

Here is an example of getting the auth token. A post with curl looks like:

```
% curl -s -u admin@localhost:changeme -v -X POST -d'{"requestType":"LOGIN"}' http://localhost:8000/core/authn/basic -H "Content-Type: application/json"
*   Trying 127.0.0.1...
* Connected to localhost (127.0.0.1) port 8000 (#0)
* Server auth using Basic with user 'admin@localhost'
> POST /core/authn/basic HTTP/1.1
> Host: localhost:8000
> Authorization: Basic YWRtaW5AbG9jYWxob3N0OmNoYW5nZW1l
> User-Agent: curl/7.43.0
> Accept: */*
> Content-Type: application/json
> Content-Length: 23
> 
* upload completely sent off: 23 out of 23 bytes
< HTTP/1.1 200 OK
< content-type: application/json
< content-length: 23
< x-xenon-auth-token: eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJkY3AiLCJzdWIiOiIvY29yZS9hdXRoei91c2Vycy9hOTk2ODIyYi1iMmRmLTRhM2QtYTk2Yi1mNTYwY2RmOWYxMzQiLCJleHAiOjE0NTIwNDIzNjk0MjYwMDB9.VACf4Xw3fbBtbKEzIRlw0KyPXMj3QYU-KwQV5kUQqjE
< set-cookie: xenon-auth-cookie=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJkY3AiLCJzdWIiOiIvY29yZS9hdXRoei91c2Vycy9hOTk2ODIyYi1iMmRmLTRhM2QtYTk2Yi1mNTYwY2RmOWYxMzQiLCJleHAiOjE0NTIwNDIzNjk0MjYwMDB9.VACf4Xw3fbBtbKEzIRlw0KyPXMj3QYU-KwQV5kUQqjE; Path=/; Max-Age=3599
< connection: keep-alive
< 
* Connection #0 to host localhost left intact
{"requestType":"LOGIN"}
```

The output is a bit complicated: look for the "x-xenon-auth-token" header above. 

If you don't mind using command-line tools to extract the cookie, you can do something like this:

```
% curl -s --stderr -  -u admin@localhost:changeme -v -X POST -d'{"requestType":"LOGIN"}' http://localhost:8000/core/authn/basic -H "Content-Type: application/json" | grep x-xenon-auth-token | awk '{print $3}'

eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJkY3AiLCJzdWIiOiIvY29yZS9hdXRoei91c2Vycy9hOTk2ODIyYi1iMmRmLTRhM2QtYTk2Yi1mNTYwY2RmOWYxMzQiLCJleHAiOjE0NTIwNDI0ODExNzkwMDF9.NrEHGi49id5f9FyYzZqLFl_54WG5OuFX2BsrxkFU3cg
```

You'll provide the the token as a header on future operations, like this:

```
% curl http://localhost:8000/core/authz/users -H "x-xenon-auth-token: eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJkY3AiLCJzdWIiOiIvY29yZS9hdXRoei91c2Vycy9hOTk2ODIyYi1iMmRmLTRhM2QtYTk2Yi1mNTYwY2RmOWYxMzQiLCJleHAiOjE0NTIwNDI0ODExNzkwMDF9.NrEHGi49id5f9FyYzZqLFl_54WG5OuFX2BsrxkFU3cg"
{
  "documentLinks": [
    "/core/authz/users/13e016c6-d69e-41f0-9ffc-e7e9939227a0",
    "/core/authz/users/a996822b-b2df-4a3d-a96b-f560cdf9f134"
  ],
  "documentCount": 2,
  "queryTimeMicros": 998,
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "6682ac24-fac5-4ff3-b9ae-66db2eb3fa3f"
}
```

If you try this command, you will need to substitute your own auth token. Note that if you try this command with a token for the other user, example@localhost, it will act in the same way as an unauthenticated user: the documentLinks list will be empty because the example user has not been authorized to GET user services. 

The token makes the command rather long, so we'll simplify commands below by replacing the auth token with the text AUTH-TOKEN. Just replace AUTH-TOKEN with whatever token you received. For example:

```
% curl http://localhost:8000/core/authz/users -H "x-xenon-auth-token: AUTH-TOKEN"
{
  "documentLinks": [
    "/core/authz/users/13e016c6-d69e-41f0-9ffc-e7e9939227a0",
    "/core/authz/users/a996822b-b2df-4a3d-a96b-f560cdf9f134"
  ],
  "documentCount": 2,
  "queryTimeMicros": 998,
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "6682ac24-fac5-4ff3-b9ae-66db2eb3fa3f"
}
```

Today, auth tokens are time-limited to one hour. That expiration time will be configurable in the future. If a token is expired, it will not work. You can request new tokens as needed: you do not need to wait until your current token expires. 

## 3.3 Authorization: How does authorization work?
How does Xenon know if a user can access a service? The answer is a combination of users, user groups, resource groups, and roles. Let's define these, then look at them. 

* __User:__ A user is a service. In the service document, a user is identified by their email.
* __User Group:__ A user group is a set of users. Access to services will be granted to user groups, not users. A user group is defined by a query. If you haven't read the [Introduction to Service Queries](./Introduction-to-Service-Queries), you'll want to do that now. 
* __Resource Group:__ A resource group is a set of service. Like user groups, resource groups are defined by queries.
* __Role:__ A role give permissions (a policy) to a single user group for a single resource group.

That might sound a little abstract, so let's look at these services for the environment we just set up with the ExampleServiceHost. 

### 3.3.1 Users
Here are two users we have made: admin@localhost and example@localhost. Other than the standard fields (documentVersion, documentKind, etc), the users only have one unique field: email. 

```
% curl 'http://localhost:8000/core/authz/users?expand' -H "x-xenon-auth-token: AUTH-TOKEN"
{
  "documentLinks": [
    "/core/authz/users/13e016c6-d69e-41f0-9ffc-e7e9939227a0",
    "/core/authz/users/a996822b-b2df-4a3d-a96b-f560cdf9f134"
  ],
  "documents": {
    "/core/authz/users/a996822b-b2df-4a3d-a96b-f560cdf9f134": {
      "email": "admin@localhost",
      "documentVersion": 0,
      "documentEpoch": 0,
      "documentKind": "com:vmware:xenon:services:common:UserService:UserState",
      "documentSelfLink": "/core/authz/users/a996822b-b2df-4a3d-a96b-f560cdf9f134",
      "documentUpdateTimeMicros": 1452037260503005,
      "documentUpdateAction": "POST",
      "documentExpirationTimeMicros": 0,
      "documentOwner": "6682ac24-fac5-4ff3-b9ae-66db2eb3fa3f",
      "documentAuthPrincipalLink": "/core/authz/system-user",
      "documentTransactionId": ""
    },
    "/core/authz/users/13e016c6-d69e-41f0-9ffc-e7e9939227a0": {
      "email": "example@localhost",
      "documentVersion": 0,
      "documentEpoch": 0,
      "documentKind": "com:vmware:xenon:services:common:UserService:UserState",
      "documentSelfLink": "/core/authz/users/13e016c6-d69e-41f0-9ffc-e7e9939227a0",
      "documentUpdateTimeMicros": 1452037260503004,
      "documentUpdateAction": "POST",
      "documentExpirationTimeMicros": 0,
      "documentOwner": "6682ac24-fac5-4ff3-b9ae-66db2eb3fa3f",
      "documentAuthPrincipalLink": "/core/authz/system-user",
      "documentTransactionId": ""
    }
  },
  "documentCount": 2,
  "queryTimeMicros": 997,
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "6682ac24-fac5-4ff3-b9ae-66db2eb3fa3f"
}
```

### 3.3.2 User Groups
We've defined two user groups. Each group is defined by a query, but the query is targeted at a single user. We've removed the standard fields from the document to focus on the essential portions of the user group:

```
% curl 'http://localhost:8000/core/authz/user-groups?expand' -H "x-xenon-auth-token: AUTH-TOKEN"
{
  "documentLinks": [
    "/core/authz/user-groups/ef9c5dea-ced1-4375-9ce3-6b5c0216ac12",
    "/core/authz/user-groups/86b928df-261a-48ff-9d56-eff01c4e75eb"
  ],
  "documents": {
    "/core/authz/user-groups/ef9c5dea-ced1-4375-9ce3-6b5c0216ac12": {
      "query": {
        "occurance": "MUST_OCCUR",
        "term": {
          "propertyName": "documentSelfLink",
          "matchValue": "/core/authz/users/13e016c6-d69e-41f0-9ffc-e7e9939227a0",
          "matchType": "TERM"
        }
      },
    ... output trimmed ...
    },
    "/core/authz/user-groups/86b928df-261a-48ff-9d56-eff01c4e75eb": {
      "query": {
        "occurance": "MUST_OCCUR",
        "term": {
          "propertyName": "documentSelfLink",
          "matchValue": "/core/authz/users/a996822b-b2df-4a3d-a96b-f560cdf9f134",
          "matchType": "TERM"
        }
      },
    ... output trimmed ...
    }
  },
  ... output trimmed ...
}
```

Here you can see two user groups, and each one is a query that will return exactly one user. If you wanted a query that included all users, you could use a query similar to this:

```
"query": {
  "occurance": "MUST_OCCUR",
  "term": {
    "propertyName": "documentSelfLink",
    "matchValue": "/core/authz/users/*",
    "matchType": "WILDCARD"
  }
}
```

### 3.3.3 Resource Groups
Resource groups are similar to user groups in that they are defined by a query. So far, our users have been defined similarly: they have similar definitions for the user and user group. Our resource groups are different. Let's take a look: (The output has been trimmed again to focus on the essentials.)

```
% curl 'http://localhost:8000/core/authz/resource-groups?expand' -H "x-xenon-auth-token: AUTH-TOKEN"
{
  "documentLinks": [
    "/core/authz/resource-groups/d7eea0b5-d814-4d63-8e29-68b32dbb6995",
    "/core/authz/resource-groups/406c1eae-54ad-4313-b0ff-a7a3075c905a"
  ],
  "documents": {
    "/core/authz/resource-groups/d7eea0b5-d814-4d63-8e29-68b32dbb6995": {
      "query": {
        "occurance": "MUST_OCCUR",
        "booleanClauses": [
          {
            "occurance": "MUST_OCCUR",
            "term": {
              "propertyName": "documentAuthPrincipalLink",
              "matchValue": "/core/authz/users/13e016c6-d69e-41f0-9ffc-e7e9939227a0",
              "matchType": "TERM"
            }
          },
          {
            "occurance": "MUST_OCCUR",
            "term": {
              "propertyName": "documentKind",
              "matchValue": "com:vmware:xenon:services:common:ExampleService:ExampleServiceState",
              "matchType": "TERM"
            }
          }
        ]
      },
    ... output trimmed ...
    },
    "/core/authz/resource-groups/406c1eae-54ad-4313-b0ff-a7a3075c905a": {
      "query": {
        "occurance": "MUST_OCCUR",
        "term": {
          "propertyName": "documentSelfLink",
          "matchValue": "*",
          "matchType": "WILDCARD"
        }
      },
    ... output trimmed ...
    }
  },
  ... output trimmed ...
}
```

The first resource group in the list (d7eea0b5...) is defined by a query with two clauses. Rephrasing them, they will find the services for which the following are true:

1. The `documentAuthPrincipalLink` is `/core/authz/users/13e016c6-d69e-41f0-9ffc-e7e9939227a0`. That is our example user.
1. The `documentKind` is `com:vmware:xenon:services:common:ExampleService:ExampleServiceState`. These will be example service documents, at /core/examples.

That is, this resource groups is "all example services owned by the example user".

The second resource group in the list (406c1eae...) is defined by a simpler query with one clause. Rephrasing it, they will find the services for which the following is true:

1. There is a `documentSelfLink`

All services have a documentSelfLink, so this resource group will specify all services. TThis resource group will be used for the admin user to provide access to all services.

_Aside:_ You might think that Xenon will evaluate this query ("find all services") then match see if the desired service is in that list. In fact, the query is used as a filter and is efficient. 

### 3.3.4 Roles
A role says: A user group X may do some set of operations (e.g. GET, POST, PATCCH) on the resources defined by a resource group. Let's look at the resource groups we have: 

```
% curl 'http://localhost:8000/core/authz/roles?expand' -H "x-xenon-auth-token: AUTH-TOKEN"
{
  "documentLinks": [
    "/core/authz/roles/db12ffe9-9245-4314-98b0-93de5fd6d068",
    "/core/authz/roles/7ce7c6d7-6d40-41da-842c-6af40fd67d32"
  ],
  "documents": {
    "/core/authz/roles/db12ffe9-9245-4314-98b0-93de5fd6d068": {
      "userGroupLink": "/core/authz/user-groups/86b928df-261a-48ff-9d56-eff01c4e75eb",
      "resourceGroupLink": "/core/authz/resource-groups/406c1eae-54ad-4313-b0ff-a7a3075c905a",
      "verbs": [
        "POST",
        "DELETE",
        "GET",
        "PATCH",
        "PUT",
        "OPTIONS"
      ],
      "policy": "ALLOW",
      "priority": 0,
      ... output trimmed ...
    },
    "/core/authz/roles/7ce7c6d7-6d40-41da-842c-6af40fd67d32": {
      "userGroupLink": "/core/authz/user-groups/ef9c5dea-ced1-4375-9ce3-6b5c0216ac12",
      "resourceGroupLink": "/core/authz/resource-groups/d7eea0b5-d814-4d63-8e29-68b32dbb6995",
      "verbs": [
        "POST",
        "DELETE",
        "GET",
        "PATCH",
        "PUT",
        "OPTIONS"
      ],
      "policy": "ALLOW",
      "priority": 0,
      ... output trimmed ...
    }
  },
  ... output trimmed ...
}
```

These two groups look similar but they are connecting different user groups and resources. The first role (db12ffe9...) is for the admin user: it connects the admin's user group with the resource group that specifies all resources. The other role is for the example user. 

### 3.3.5 Recap

To recap: Xenon defines authorization via roles. A role will grant permission to do a set of operations (GET, POST, etc) on a set of resources (a resource group) to a set of users (a user group). Resource groups and user groups are defined by queries. 

At first look, defining resource and user groups as queries may seem more complicated than as simple lists. However, it can allow for flexible and simple statements about access to services in Xenon. 

# 4 Authentication and Authorization in Java

The example Java code below is from [AuthorizationSetupHelper.java](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/common/AuthorizationSetupHelper.java), which is used by the [ExampleServiceHost](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/ExampleServiceHost.java) to create the admin and example user we showed above. 

If you're browsing the source code, the following table will help you find the right services. 

Service | URI | Java code
--------|-----|-----------
Users | /core/authz/users | [UserService.java](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/UserService.java)
User Groups | /core/authz/user-groups | [UserGroupService.java](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/UserGroupService.java)
Resource Groups | /core/authz/resource-groups | [ResourceGroupService.java](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/ResourceGroupService.java)
Roles | /core/authz/roles | [RoleService.java](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/RoleService.java)
Basic Auth | /core/authn/basic | [BasicAuthenticationService.java](https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/services/common/authn/BasicAuthenticationService.java)

All of the example code below has been extended with extra comments to help explain it in detail.

## 4.1 Creating a user

This code creates a user. It a simple POST to the user service factory. The body of the POST describes the user with just one field, the email address.

```java
private void makeUser() {
    // Describe the user: just an email address
    UserState user = new UserState();
    user.email = this.userEmail;

    // Make a POST to the user factory service
    URI userFactoryUri = UriUtils.buildUri(this.host, ServiceUriPaths.CORE_AUTHZ_USERS);
    Operation postUser = Operation.createPost(userFactoryUri)
            .setBody(user)
            .setReferer(this.referer)
            .setCompletion((op, ex) -> {
                if (ex != null) {
                    this.failureMessage = String.format("Could not make user %s: %s",
                            this.userEmail, ex);
                    this.currentStep = UserCreationStep.FAILURE;
                    setupUser();
                    return;
                }
                // Extract the self link made by the user service
                UserState userResponse = op.getBody(UserState.class);
                this.userSelfLink = userResponse.documentSelfLink;
                this.currentStep = UserCreationStep.MAKE_CREDENTIALS;
                setupUser();
            });
    this.host.sendRequest(postUser);
}
```

# 4.2 Creating user credentials

User credentials are POST'ed to the basic authentication service. In the future when integration with ActiveDirectory and other authentication services are provided, this will not be done through Xenon. 

```java
private void makeCredentials() {
    // Describe the credentials for the basic authentication service
    AuthCredentialsServiceState auth = new AuthCredentialsServiceState();
    auth.userEmail = this.userEmail;
    auth.privateKey = this.userPassword;

    // POST the credentials to the basic authentication service
    URI credentialFactoryUri = UriUtils.buildUri(this.host, ServiceUriPaths.CORE_CREDENTIALS);
    Operation postCreds = Operation.createPost(credentialFactoryUri)
            .setBody(auth)
            .setReferer(this.referer)
            .setCompletion((op, ex) -> {
                if (ex != null) {
                    this.failureMessage = String.format("Could not make credentials for user %s: %s",
                            this.userEmail, ex);
                    this.currentStep = UserCreationStep.FAILURE;
                    setupUser();
                    return;
                }
                this.currentStep = UserCreationStep.MAKE_USER_GROUP;
                setupUser();
        });
    this.host.sendRequest(postCreds);
}
```

## 4.3 Create a user group

This creates a user group by making a POST to the user group factory. The body of the POST has just one field, the query that specifies the user group.

``` java
private void makeUserGroup() {
    // Make a query that will return exactly one user, the one we created
    Query userQuery = Query.Builder.create()
            .setTerm(ServiceDocument.FIELD_NAME_SELF_LINK, this.userSelfLink)
            .build();

    // Create a user group that uses that query
    UserGroupState group = new UserGroupState();
    group.query = userQuery;

    // POST the user group to the user group service
    URI userGroupFactoryUri = UriUtils.buildUri(this.host,
            ServiceUriPaths.CORE_AUTHZ_USER_GROUPS);
    Operation postGroup = Operation.createPost(userGroupFactoryUri)
            .setBody(group)
            .setReferer(this.referer)
            .setCompletion((op, ex) -> {
                if (ex != null) {
                    this.failureMessage = String.format("Could not make user group for user %s: %s",
                            this.userEmail, ex);
                    this.currentStep = UserCreationStep.FAILURE;
                    setupUser();
                    return;
                }
                // Extract the user group self link to use when we make the role
                UserGroupState groupResponse = op.getBody(UserGroupState.class);
                this.userGroupSelfLink = groupResponse.documentSelfLink;
                this.currentStep = UserCreationStep.MAKE_RESOURCE_GROUP;
                setupUser();
            });
    this.host.sendRequest(postGroup);
}
```

## 4.4 Create a resource group

This code is a bit more complicated because it can create two kinds of resource groups, based on whether we're making an admin user or the example user. In each case, the resource group is made with a POST to the resource group factory service and the body of the POST contains just the query used to specify the group.

```java
private void makeResourceGroup() {
    Query resourceQuery;
     if (this.isAdmin) {
        // Make a query that matches all services.
        // The query is "selfLink == *", where * is the wildcard
        resourceQuery = Query.Builder.create()
                .setTerm(ServiceDocument.FIELD_NAME_SELF_LINK, UriUtils.URI_WILDCARD_CHAR,
                        QueryTerm.MatchType.WILDCARD)
                .build();
    } else {
        // Make a query that matches all services of one type (example services in this tutorial)
        // owned by one user (example user in this tutorial)
        resourceQuery = Query.Builder.create()
                .addFieldClause(ServiceDocument.FIELD_NAME_AUTH_PRINCIPAL_LINK,
                        this.userSelfLink)
                .addFieldClause(ServiceDocument.FIELD_NAME_KIND, this.documentKind)
                .build();
    }
    ResourceGroupState group = new ResourceGroupState();
    group.query = resourceQuery;
     URI resourceGroupFactoryUri = UriUtils.buildUri(this.host,
	    ServiceUriPaths.CORE_AUTHZ_RESOURCE_GROUPS);
    Operation postGroup = Operation.createPost(resourceGroupFactoryUri)
	    .setBody(group)
	    .setReferer(this.referer)
	    .setCompletion((op, ex) -> {
                if (ex != null) {
		    this.failureMessage = String.format("Could not make resource group for user %s: %s",
			    this.userEmail, ex);
		    this.currentStep = UserCreationStep.FAILURE;
		    setupUser();
		    return;
                }

                // Extract the resource group self link so we can use it to make the role
                ResourceGroupState groupResponse = op.getBody(ResourceGroupState.class);
                this.resourceGroupSelfLink = groupResponse.documentSelfLink;
                this.currentStep = UserCreationStep.MAKE_ROLE;
                setupUser();
	    });
    this.host.sendRequest(postGroup);
}
```

## 4.5 Make the role

Finally, we make the role by making a POST to the role factory service. The body of the role has the self links of the user and resource groups.

```java
private void makeRole() {
    // Define the set of operations (GET, POST, etc). We'll use "all operations"
    Set<Action> verbs = new HashSet<>();                                                                             
    for (Action action : Action.values()) {
        verbs.add(action);                                                                                           
    }

    // Define the role using the user and resource group previously created
    RoleState role = new RoleState();                                                                                
    role.userGroupLink = this.userGroupSelfLink;                                                                     
    role.resourceGroupLink = this.resourceGroupSelfLink;                                                             
    role.verbs = verbs;                                                                                              
    role.policy = Policy.ALLOW;                                                                                      
     URI resourceGroupFactoryUri = UriUtils.buildUri(this.host,
	    ServiceUriPaths.CORE_AUTHZ_ROLES);                                                                       
    Operation postGroup = Operation.createPost(resourceGroupFactoryUri)
	    .setBody(role)
	    .setReferer(this.referer)
	    .setCompletion((op, ex) -> {
                if (ex != null) {
		    this.failureMessage = String.format("Could not make role for user %s: %s",
			    this.userEmail, ex);                                                                     
		    this.currentStep = UserCreationStep.FAILURE;                                                     
		    setupUser();                                                                                     
		    return;                                                                                          
                }

                // Extract the self link so we can report it in the log
                RoleState roleResponse = op.getBody(RoleState.class);                                                
                this.roleSelfLink = roleResponse.documentSelfLink;                                                   
                this.currentStep = UserCreationStep.SUCCESS;                                                         
                setupUser();                                                                                         
	    });                                                                                                      
    this.host.sendRequest(postGroup);                                                                                
}
```

# 5.0 Commonly asked questions

## Is it possible to have authentication without authorization?
No: authentication and authorization go hand-in-hand. Note that you create users with full access to all services, as we did above.

## Can non-authenticated and non-authorized users access core services?
No. See the examples above.

# 6.0 More Information
* [./Authentication-and-Authorization-Design](Authentication and Authorization Design)
