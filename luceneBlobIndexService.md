# Overview

The blob index service uses a lucene index, isolated from the document index, to store binary content. The content can be retrieved by a primary key. Unlike the document index, it does not attempt to index the blob, its treated as a bag of bytes. It also currently supports only a single version per key.

# REST API

## URI
```
/core/blob-index
```
## POST

A client can insert a blob using the following HTTP request
```
POST /core/blob-index?key=some-key&updateTime=timestamp
```
A service can create POST operations using the following helper:
```
LuceneBlobIndexService.createPost(this.host, key, objectToInsert);
```
## GET

A client has to be co-located (in the same process) as the blob index to query the index. This restriction can be easily lifted in the future. A service can request a previously inserted blob by using the helper methods in LuceneBlobIndexService:
```
LuceneBlobIndexService.createGet(this.host, key);
```