# 1.0 Overview

This page gives you an introduction to querying services. This is a fundamental concept used throughout Xenon. For example, authorization in Xenon is based on queries. 

This tutorial assumes that you have gone through the introductory [Example Service Tutorial](Example-Service-Tutorial). There is an in-depth [Query Task Tutorial](./QueryTaskService) that will give you more extensive detail, including making queries in Java. 

This tutorial assumes you have installed Xenon and can run it from the command-line. It also assumes you have installed the "curl" command-line tool. If you don't want to use curl, the examples should be easily converted to use another tool, such as Postman. 

# 2.0 Starting the Xenon host

We are going to start the Xenon host with an extra argument that specifies the _sandbox_. The sandbox is the directory that holds all of the storage for Xenon on a single host. (When Xenon is running with multiple hosts, each has their own sandbox.)

You are not required to do this, but it will make repeat tests easier: you can remove the directory in order to start again with an empty storage system. 

```
% java -jar xenon-host/target/xenon-host-0.4.0-SNAPSHOT-jar-with-dependencies.jar --sandbox=/tmp/xenon
[0][I][1452021498723][DecentralizedControlPlaneHost:8000][startImpl][ServiceHost/cd70e1c8 listening on 127.0.0.1:8000]
```

# 3.0 Creating some services
We are going to create four services: two examples and two users. 

## 3.1 Create Examples
First, make two files:

_File:_ example-1.body
```javascript
{
  "name": "example-1",
  "counter": 1
}
```

_File:_ example-2.body
```javascript
{
  "name": "example-2",
  "counter": 1
}
```

Then POST them to Xenon:

```sh
% curl -s -X POST -d@example-1.body http://localhost:8000/core/examples -o /dev/null -H "Content-Type: application/json" 
% curl -s -X POST -d@example-2.body http://localhost:8000/core/examples -o /dev/null -H "Content-Type: application/json"
```

## 3.2 Create Users
Now, make two files for the users:

_File:_ user-1.body
```javascript
{
  "email": "user1@example.com",
  "documentKind": "com:vmware:xenon:services:common:UserService:UserState"
}
```

_File:_ user-2.body
```javascript
{
  "email": "user2@example.com",
  "documentKind": "com:vmware:xenon:services:common:UserService:UserState"
}
```

Then post them to Xenon:
```sh
% curl -s  -X POST -d@user-1.body http://localhost:8000/core/authz/users -o /dev/null -H "Content-Type: application/json"
% curl -s  -X POST -d@user-2.body http://localhost:8000/core/authz/users -o /dev/null -H "Content-Type: application/json"
```

# 4.0 Querying the services directly

You can query the factory services to see what services they have. For instance, to query for all the user documents, you can do:

```sh
% curl http://localhost:8000/core/examples
{
  "documentLinks": [
    "/core/examples/a94a1f1c-acdf-4b18-97f4-35850f48eb0c",
    "/core/examples/bc1a0009-e7bc-4efa-b823-57ada40c02d4"
  ],
  "documentCount": 2,
  "queryTimeMicros": 1000,
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "c0c8e4ac-638d-4ad9-810a-b1c5250275aa"
}
```

Note that this only gave you _links_ to the documents. This is efficient when there are a large number of documents. 

You may prefer to see the complete documents. You can do this with the _expand_ option:

```sh
% curl 'http://localhost:8000/core/examples?$expand'
{
  "documentLinks": [
    "/core/examples/a94a1f1c-acdf-4b18-97f4-35850f48eb0c",
    "/core/examples/bc1a0009-e7bc-4efa-b823-57ada40c02d4"
  ],
  "documents": {
    "/core/examples/bc1a0009-e7bc-4efa-b823-57ada40c02d4": {
      "keyValues": {},
      "counter": 2,
      "name": "example-2",
      "documentVersion": 0,
      "documentEpoch": 0,
      "documentKind": "com:vmware:xenon:services:common:ExampleService:ExampleServiceState",
      "documentSelfLink": "/core/examples/bc1a0009-e7bc-4efa-b823-57ada40c02d4",
      "documentUpdateTimeMicros": 1452022631227002,
      "documentUpdateAction": "POST",
      "documentExpirationTimeMicros": 0,
      "documentOwner": "c0c8e4ac-638d-4ad9-810a-b1c5250275aa",
      "documentAuthPrincipalLink": "/core/authz/guest-user",
      "documentTransactionId": ""
    },
    "/core/examples/a94a1f1c-acdf-4b18-97f4-35850f48eb0c": {
      "keyValues": {},
      "counter": 1,
      "name": "example-1",
      "documentVersion": 0,
      "documentEpoch": 0,
      "documentKind": "com:vmware:xenon:services:common:ExampleService:ExampleServiceState",
      "documentSelfLink": "/core/examples/a94a1f1c-acdf-4b18-97f4-35850f48eb0c",
      "documentUpdateTimeMicros": 1452022286429000,
      "documentUpdateAction": "POST",
      "documentExpirationTimeMicros": 0,
      "documentOwner": "c0c8e4ac-638d-4ad9-810a-b1c5250275aa",
      "documentAuthPrincipalLink": "/core/authz/guest-user",
      "documentTransactionId": ""
    }
  },
  "documentCount": 2,
  "queryTimeMicros": 1,
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "c0c8e4ac-638d-4ad9-810a-b1c5250275aa"
}
```
