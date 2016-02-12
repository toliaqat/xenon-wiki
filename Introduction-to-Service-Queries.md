# 1.0 Overview

This page gives you an introduction to querying services. This is a fundamental concept used throughout Xenon. For example, authorization in Xenon is based on queries. 

This tutorial assumes that you have gone through the introductory [Example Service Tutorial](Example-Service-Tutorial). There is an in-depth description of queries in the [Query Task Service](./QueryTaskService) that will give you more detail, including making queries in Java. 

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

You can also query the individual documents directly:

```sh
% curl http://localhost:8000/core/examples/a94a1f1c-acdf-4b18-97f4-35850f48eb0c
{
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
```

# 5.0 Querying the Document Index
Internally, all service documents are stored in Lucene and are managed by the Lucene index service. When you query a factory service, it simply translates that to a query to the Lucene index service. Although you would probably never directly query the Lucene index directly, it's instructive to do so. 

If you directly query the Lucene query, it will complain:

```sh
% curl http://localhost:8000/core/document-index
{
  "message": "documentSelfLink query parameter is required",
  "statusCode": 400,
  "documentKind": "com:vmware:xenon:common:ServiceErrorResponse"
}
```

If you give it a link to a service, it will return the document: 

```sh
% curl 'http://localhost:8000/core/document-index?documentSelfLink=/core/examples/a94a1f1c-acdf-4b18-97f4-35850f48eb0c'
{
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
```

You can include wildcards in your link. This will return all example service documents: 

```sh
% curl 'http://localhost:8000/core/document-index?documentSelfLink=/core/examples/*'
{
  "documentLinks": [
    "/core/examples/a94a1f1c-acdf-4b18-97f4-35850f48eb0c",
    "/core/examples/bc1a0009-e7bc-4efa-b823-57ada40c02d4"
  ],
  "documentCount": 2,
  "queryTimeMicros": 1998,
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "c0c8e4ac-638d-4ad9-810a-b1c5250275aa"
}
```

You can expand them to see the full documents. Note that we've trimmed the output for simplicity, but the full documents are returned.
```sh
% curl 'http://localhost:8000/core/document-index?documentSelfLink=/core/examples/*&expand=documentLinks'
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
      ... output trimmed ...
    },
    "/core/examples/a94a1f1c-acdf-4b18-97f4-35850f48eb0c": {
      "keyValues": {},
      "counter": 1,
      "name": "example-1",
      ... output trimmed ...
    }
  },
  "documentCount": 2,
  "queryTimeMicros": 999,
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "c0c8e4ac-638d-4ad9-810a-b1c5250275aa"
}
```

You can even see all the documents in the index:

```sh
% curl 'http://localhost:8000/core/document-index?documentSelfLink=/*'
{
  "documentLinks": [
    "/core/examples/a94a1f1c-acdf-4b18-97f4-35850f48eb0c",
    "/core/authz/users/47f7449f-0c38-4d18-a2a2-96f4378d492a",
    "/core/authz/users/e948c9e2-8812-4469-af01-580b2579930e",
    "/core/examples/bc1a0009-e7bc-4efa-b823-57ada40c02d4"
  ],
  "documentCount": 4,
  "queryTimeMicros": 998,
  "documentVersion": 0,
  "documentUpdateTimeMicros": 0,
  "documentExpirationTimeMicros": 0,
  "documentOwner": "c0c8e4ac-638d-4ad9-810a-b1c5250275aa"
}
```

# 6.0 Query Tasks

## 6.1 Query tasks: the client view
As nice as the above queries are, they are limited. You may need to do full boolean queries, such as "as example documents owned by user-1@example.com". For this, you need query tasks. 

Here is one example of a simple query task. All query tasks are done a a POST. A _direct_ query will return the results in response to the POST. If you do not specify that it is a direct query, the results can be queried later; this is useful when the query may take a long time to execute or is a continuous query. 

For this query, we specify the query in a separate file:

_File:_ query-all.body
```javascript
{
  "taskInfo": {
    "isDirect": true
  },
  "querySpec": {
    "query": {
      "term": {
        "propertyName": "documentKind",
        "matchValue": "*",
        "matchType": "WILDCARD"
      }
    }
  },
  "indexLink": "/core/document-index"
}
```

Here is how we use the query. _Please note:_ We are using the [jq tool](https://stedolan.github.io/jq/) in order to format the output. If you do not have this tool, remove it's use and you'll see an unformatted response.

```sh
% curl -s -X POST -d@query-all.body http://localhost:8000/core/query-tasks -H "Content-Type: application/json" | jq .
{
  "taskInfo": {
    "stage": "FINISHED",
    "isDirect": true,
    "durationMicros": 1
  },
  "querySpec": {
    "query": {
      "occurance": "MUST_OCCUR",
      "term": {
        "propertyName": "documentKind",
        "matchValue": "*",
        "matchType": "WILDCARD"
      }
    },
    "resultLimit": 2147483647,
    "options": []
  },
  "results": {
    "documentLinks": [
      "/core/examples/a94a1f1c-acdf-4b18-97f4-35850f48eb0c",
      "/core/authz/users/47f7449f-0c38-4d18-a2a2-96f4378d492a",
      "/core/authz/users/e948c9e2-8812-4469-af01-580b2579930e",
      "/core/examples/bc1a0009-e7bc-4efa-b823-57ada40c02d4"
    ],
    "documentCount": 4,
    "queryTimeMicros": 1,
    "documentVersion": 0,
    "documentUpdateTimeMicros": 0,
    "documentExpirationTimeMicros": 0,
    "documentOwner": "c0c8e4ac-638d-4ad9-810a-b1c5250275aa"
  },
  "indexLink": "/core/document-index",
  "documentVersion": 0,
  "documentEpoch": 0,
  "documentKind": "com:vmware:xenon:services:common:QueryTask",
  "documentSelfLink": "/core/query-tasks/f490f4a3-3292-4400-88e8-5fb750af4de6",
  "documentUpdateTimeMicros": 1452023742326002,
  "documentExpirationTimeMicros": 1452024342326005,
  "documentOwner": "c0c8e4ac-638d-4ad9-810a-b1c5250275aa",
  "documentAuthPrincipalLink": "/core/authz/guest-user"
}
```

Here is one more example, to query all users:

_File:_ query-users.body
```
{
  "taskInfo": {
    "isDirect": true
  },
  "querySpec": {
    "query": {
      "term": {
        "propertyName": "documentKind",
        "matchValue": "com:vmware:xenon:services:common:UserService:UserState",
        "matchType": "TERM"
      }
    }
  },
  "indexLink": "/core/document-index"
}
```

Using the query (output trimmed for simplicity):

```
% curl -s -X POST -d@query-users.body http://localhost:8000/core/query-tasks -H "Content-Type: application/json" | jq . 
{
  "taskInfo": {
    "stage": "FINISHED",
    "isDirect": true,
    "durationMicros": 1
  },
  "querySpec": {
    "query": {
      "occurance": "MUST_OCCUR",
      "term": {
        "propertyName": "documentKind",
        "matchValue": "com:vmware:xenon:services:common:UserService:UserState",
        "matchType": "TERM"
      }
    },
    "resultLimit": 2147483647,
    "options": []
  },
  "results": {
    "documentLinks": [
      "/core/authz/users/47f7449f-0c38-4d18-a2a2-96f4378d492a",
      "/core/authz/users/e948c9e2-8812-4469-af01-580b2579930e"
    ],
  ... output trimmed ...
  },
  "indexLink": "/core/document-index",
  ... output trimmed ...
}
```

## 6.2 Query Tasks in Java
Let's look at an example of using Query Tasks in Java within Xenon. Our goal is to find out if there are any users with the email address of user1@example.com. To do this, we need to do a query for all user service documents with an email field of "user1@example.com". This is a boolean AND query. Let's first look at it in the command-line:

_File:_ query-user-1.body
```
{
  "taskInfo": {
    "isDirect": true
  },
  "querySpec": {
    "query": {
      "booleanClauses": [
         {
           "occurance": "MUST_OCCUR",
           "term": {
             "propertyName": "documentKind",
             "matchValue": "com:vmware:xenon:services:common:UserService:UserState",
             "matchType": "TERM"
           }
         },
         {
           "occurance": "MUST_OCCUR",
           "term": {
             "propertyName": "email",
             "matchValue": "user1@example.com",
             "matchType": "TERM"
           }
         }
      ]
    }
  },
  "indexLink": "/core/document-index"
}
```

Note that this query has "booleanClauses" instead of "term", and each element in the booleanClauses is a single term. Each term could itself be booleanClause instead. 

Let's try the query out (output trimmed for simplicity):
```
% curl -s -X POST -d@query-user-1.body http://localhost:8000/core/query-tasks -H "Content-Type: application/json" | jq . 
{
  "taskInfo": {
    "stage": "FINISHED",
    "isDirect": true,
    "durationMicros": 992
  },
  "querySpec": {
    "query": {
      "occurance": "MUST_OCCUR",
      ... output trimmed ... 
    },
    "resultLimit": 2147483647,
    "options": []
  },
  "results": {
    "documentLinks": [
      "/core/authz/users/47f7449f-0c38-4d18-a2a2-96f4378d492a"
    ],
   ... output trimmed ... 
  },
  "indexLink": "/core/document-index",
   ... output trimmed ... 
}
```

Excellent: we got exactly the one service document that has the user we are looking for. 

Now let's try it in Java. This query is taken from [https://github.com/vmware/xenon/blob/master/xenon-common/src/main/java/com/vmware/xenon/common/AuthorizationSetupHelper.java](AuthorizationSetupHelper.java). That code is doing a query to see if a user exists before it attempts to create the user. Let's look at the Java code, annotated with a few extra comments to explain the code

```java
private void queryUser() {
    // This is the query that will correspond to the "query" portion of query-user-1.body, 
    // above. Note that we don't specify MUST_OCCUR because it's the default
    Query userQuery = Query.Builder.create()
            // This is the first of the boolean clauses, to match all user service documents
            // addFieldClause will always populate the booleanClauses field, not the term field
            .addFieldClause(ServiceDocument.FIELD_NAME_KIND, Utils.buildKind(UserState.class))

            // This is the second of the boolean clauses, to match just the email address
            .addFieldClause(UserState.FIELD_NAME_EMAIL, this.userEmail)
            .build();

    // This will correspond to the entire query-user-1.body, above. It just embeds the 
    // query we created above.
    QueryTask queryTask = QueryTask.Builder.createDirectTask()
            .setQuery(userQuery)
            .build();

    URI queryTaskUri = UriUtils.buildUri(this.host, ServiceUriPaths.CORE_QUERY_TASKS);

    // This will create the POST that corresponds to the query-user-1.post, above
    Operation postQuery = Operation.createPost(queryTaskUri)
            .setBody(queryTask)
            .setReferer(this.referer)
            .setCompletion((op, ex) -> {
                if (ex != null) {
                    this.failureMessage = String.format("Could not query user %s: %s",
                            this.userEmail, ex);
                    this.currentStep = UserCreationStep.FAILURE;
                    setupUser();
                    return;
                }

                // This examines the response from the Query Task Service
                // In our case, we decide a user exists if there's at least
                // one user in the response.
                QueryTask queryResponse = op.getBody(QueryTask.class);
                if (queryResponse.results.documentLinks != null
                        && queryResponse.results.documentLinks.isEmpty()) {
                    this.currentStep = UserCreationStep.MAKE_USER;
                    setupUser();
                    return;
                }
                this.host.log(Level.INFO, "User %s already exists, skipping setup of user",
                        this.userEmail);
            });

    // This sends the request
    this.host.sendRequest(postQuery);
}
```

## 6.3 Learn more about Query Tasks
To learn more about query tasks please see the [Query Task Service](./QueryTaskService)