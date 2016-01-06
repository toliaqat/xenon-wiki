# 1.0 Overview
This page gives you an introduction to authentication and authorization in Xenon. 

This tutorial assumes that you have gone through the introductory [Example Service Tutorial](Example-Service-Tutorial) as well as the [Introduction to Service Queries](./Introduction-to-Service-Queries). Those tutorials did not enable authentication and authorization. In a production environment, you'll certainly enable them. This tutorial will help you understand how it works. 

This tutorial assumes you have installed Xenon and can run it from the command-line. It also assumes you have installed the "curl" command-line tool. If you don't want to use curl, the examples should be easily converted to use another tool, such as Postman. It also assumes you have installed the [jq tool](https://stedolan.github.io/jq/), which we use for formatting JSON responses from Xenon. If you do not have it installed, you can simply remove it from the commands below. 

# 2.0 Starting the Xenon host

We are going to start the Xenon host differently than the previous tutorials:

1. We are going to start a different host, the ExampleServiceHost. This host provides extra arguments to make it easier to work with authorization
2. We are going to provide arguments to enable authentication and authorization as well as create two users. The two users will be:
  * User "root@localhost" with password "changeme"
  * User "example@localhost" with password "changeme"

Users in Xenon are always identified by an email address. 

```sh
% java -cp xenon-host/target/xenon-host-0.4.0-SNAPSHOT-j-with-dependencies.jar com.vmware.xenon.services.common.ExampleServiceHost --sandbox=/tmp/xenon --isAuthorizationEnabled=true --adminUser=admin@localhost --adminUserPassword=changeme --exampleUser=example@localhost --exampleUserPassword=changeme

[0][I][1452037260285][ExampleServiceHost:8000][startImpl][ServiceHost/cd70e1c8 listening on 127.0.0.1:8000]
[1][I][1452037260557][ExampleServiceHost:8000][printUserDetails][Created user example@localhost (/core/authz/users/13e016c6-d69e-41f0-9ffc-e7e9939227a0) with credentials, user group (/core/authz/user-groups/ef9c5dea-ced1-4375-9ce3-6b5c0216ac12) resource group (/core/authz/resource-groups/d7eea0b5-d814-4d63-8e29-68b32dbb6995) and role(/core/authz/roles/7ce7c6d7-6d40-41da-842c-6af40fd67d32)]
[2][I][1452037260557][ExampleServiceHost:8000][printUserDetails][Created user admin@localhost (/core/authz/users/a996822b-b2df-4a3d-a96b-f560cdf9f134) with credentials, user group (/core/authz/user-groups/86b928df-261a-48ff-9d56-eff01c4e75eb) resource group (/core/authz/resource-groups/406c1eae-54ad-4313-b0ff-a7a3075c905a) and role(/core/authz/roles/db12ffe9-9245-4314-98b0-93de5fd6d068)]
```

The output is a bit long, but you can learn quite a bit from it. For example, we have created a user named admin@localhost:
* Each user is represented by a user service. This user is at: /core/authz/users/a996822b-b2df-4a3d-a96b-f560cdf9f134
* We made a user group that contains just this user. The user group is at: /core/authz/user-groups/86b928df-261a-48ff-9d56-eff01c4e75eb
* We made a resource group that contains a set of resources: /core/authz/resource-groups/406c1eae-54ad-4313-b0ff-a7a3075c905a
* We made a role that connects together the user group and resource group: /core/authz/roles/db12ffe9-9245-4314-98b0-93de5fd6d068

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

## 3.2 Getting an auth token
All API operations require an auth token. The API workflow is:

* POST user credentials to an authentication service. Currently this is /core/authn/basic, which authenticates local users based on password, but other authentication services (e.g. ActiveDirectory) will be added in the future. 
* If the credentials are correct, the user will receive an _auth token_
* Future API calls should provide the auth token as a header named "x-xenon-auth-token"

Alternatively, clients can use cookies: the auth token is embedded in the xenon-auth-cookie. The auth token header is the preferred approach, but both are acceptable. 

Here is an example of getting the auth token. A post with curl looks like:

```sh
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

```sh
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

If you try this command, you will need to substitute your own auth token.

That's rather long, so we'll simplify commands below by replacing the auth token with the text AUTH-TOKEN. Just replace AUTH-TOKEN with whatever token you received. For example:

```sh
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

Today, auth tokens are time-limited to one hour. That expiration time will be configurable in the future. 