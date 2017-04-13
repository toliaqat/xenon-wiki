# Backup/Restore

The [[HostManagementService]] provides an asynchronous API for backup/restore the document index, through internal document index service requests.

## Backup

Send a PATCH `BackupRequest` request to `/core/management`.

`BackupRequest`:
- `kind`:
  specify `com:vmware:xenon:services:common:ServiceHostManagementService:BackupRequest`
- `destination`:
  URI with `http/https` scheme or `file` scheme with local file or directory are supported.
  When `http/https` is specified, destination is expected to accept PUT request with range header.
  The PUT request contains a zip file that contains files used by document index service.
- `backupType`: `ZIP`, `DIRECTORY`, and `STREAM` is supported.  
  `ZIP` is default and generates a zipped backup local file.  
  `DIRECTORY` is supported for a local directory and performs incremental backup.  
  `STREAM` is for `http/https` destination. zipped content will be sent to the destination as PUT requests.


In the request body, `kind` and `destination` are required parameters.

*NOTE:*  
For backward compatibility, currently `backupType` parameter is optional and default to `ZIP` if not specified.
However, it is recommended to always explicitly specify this parameter since it may become required parameter in future.


Sample:

```shell
> curl -X PATCH -H "Content-Type: application/json" -d '
{
  "kind": "com:vmware:xenon:services:common:ServiceHostManagementService:BackupRequest",
  "destination": "file:/var/backup/xenon-backup.zip",
  "backupType": "ZIP"
}
' http://localhost:8000/core/management
```

### Incremental Backup

When `backupType=DIRECTORY` and local directory is specified in `destination`, it will perform incremental backup.  
Incremental backup only copies new files and deletes unused index files in new snapshot to the destination directory.  
Each incremental backup operation updates the contents of the directory to reflect the most recent state of the index.

For example, when backup is performed first time, it will simply add all index files into the destination directory.
Since index files are immutable, when next backup is performed pointing to the directory that contains previous backup files, it will copy ONLY newly created index files and remove unused files.



Sample:

```shell
curl -X PATCH -H "Content-Type: application/json" -d '
{
  "kind": "com:vmware:xenon:services:common:ServiceHostManagementService:BackupRequest",
  "destination": "file:/var/xenon-backup/",
  "backupType": "DIRECTORY"
}
' http://localhost:8010/core/management
```


## Restore

Send a PATCH `RestoreRequest` request to `/core/management`.  
**NOTE:** you **MUST** restart the service host that processed the restore request.

`RestoreRequest`:
- `kind`:
  specify `com:vmware:xenon:services:common:ServiceHostManagementService:RestoreRequest`
- `destination`:
  URI with `http/https` scheme or `file` scheme with local file or directory are supported.
  When `http/https` is specified, it is expected to accept GET requests from target xenon host with
  range header, and return a zip file that contains files used by document index service.

Sample:

```shell
> curl -X PATCH -H "Content-Type: application/json" -d '
{
  "kind": "com:vmware:xenon:services:common:ServiceHostManagementService:RestoreRequest",
  "destination": "file:/var/backup/xenon-backup.zip"
}
' http://localhost:8000/core/management
```


### Time Snapshot Boundary

When `timeSnapshotBoundaryMicros` parameter is specified in restore request, recovered data will not have documents updated after the specified time. This will make point in time recovery available from the backup.


## In-memory document index

`InMemoryLuceneDocumentIndexService` is an opt-in service to store document index in memory.  
`LuceneDocumentIndexBackupService` works for both file-based and in-memory based document index services.

To perform backup/restore to in-memory document index service, backup service needs to be initialized with in-memory index service, and register it to the host. (In example below, it uses `ServiceUriPaths.CORE_IN_MEMORY_DOCUMENT_INDEX_BACKUP` for convenience, but not limited to)


## Starting in-memory related services

```java
host.addPrivilegedService(InMemoryLuceneDocumentIndexService.class);

InMemoryLuceneDocumentIndexService inMemoryIndexService = new InMemoryLuceneDocumentIndexService();
LuceneDocumentIndexBackupService inMemoryIndexBackupService = new LuceneDocumentIndexBackupService(inMemoryIndexService);

host.startService(Operation.createPost(host, ServiceUriPaths.CORE_IN_MEMORY_DOCUMENT_INDEX), inMemoryIndexService);
host.startService(Operation.createPost(host, ServiceUriPaths.CORE_IN_MEMORY_DOCUMENT_INDEX_BACKUP), inMemoryIndexBackupService);
```

## Backup

When requesting a backup, specify `backupServiceLink` in request.

```shell
> curl -X PATCH -H "Content-Type: application/json" -d '
{
  "kind": "com:vmware:xenon:services:common:ServiceHostManagementService:BackupRequest",
  "destination": "file:/var/backup/xenon-backup.zip",
  "backupType": "ZIP",
  "backupServiceLink": "/core/in-memory-document-index-backup"
}
' http://localhost:8000/core/management
```

## Restore

To restore, specify `backupServiceLink` in request.

```shell
> curl -X PATCH -H "Content-Type: application/json" -d '
{
  "kind": "com:vmware:xenon:services:common:ServiceHostManagementService:RestoreRequest",
  "destination": "file:/var/backup/xenon-backup.zip",
  "backupServiceLink": "/core/in-memory-document-index-backup"
}
' http://localhost:8000/core/management
```

When it is restoring to the in-memory index service, host restart is not required.  
If you restart the host, in-memory data will be cleared.
