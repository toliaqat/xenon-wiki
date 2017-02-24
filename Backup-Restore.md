# Backup/Restore

The [[HostManagementService]] provides an asynchronous API for backup/restore the document index, through internal document index service requests.

## Backup

Send a PATCH `BackupRequest` request to `/core/management`.  
In the request body, `kind` and `destination` are required.

`BackupRequest`:
- `kind`:
  specify `com:vmware:xenon:services:common:ServiceHostManagementService:BackupRequest`
- `destination`:
  URI with `http/https` scheme or `file` scheme with local file are supported.
  When `http/https` is specified, destination is expected to accept PUT request with range header.
  The PUT request contains a zip file that contains files used by document index service.

Sample:

```shell
> curl -X PATCH -H "Content-Type: application/json" -d '
{
  "kind": "com:vmware:xenon:services:common:ServiceHostManagementService:BackupRequest",
  "destination": "file:/var/backup/xenon-backup.zip"
}
' http://localhost:8000/core/management
```


## Restore

Send a PATCH `RestoreRequest` request to `/core/management`.  
**NOTE:** you **MUST** restart the service host that processed the restore request.

`RestoreRequest`:
- `kind`:
  specify `com:vmware:xenon:services:common:ServiceHostManagementService:RestoreRequest`
- `destination`:
  URI with `http/https` scheme or `file` scheme with local file are supported.
  When `http/https` is specified, it is expected to accept GET request from target xenon host with
  range header, and return a zip file that contains files used by document index service.

```shell
> curl -X PATCH -H "Content-Type: application/json" -d '
{
  "kind": "com:vmware:xenon:services:common:ServiceHostManagementService:RestoreRequest",
  "destination": "file:/var/backup/xenon-backup.zip"
}
' http://localhost:8000/core/management
```


### Time Snapshot Boundary

When `timeSnapshotBoundaryMicros` parameter is specified in restore request, recovered data will not have documents updated after the speficied time. This will make point in time recovery available from the backup.
