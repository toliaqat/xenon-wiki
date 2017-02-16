# Backup/Restore

The [[HostManagementService]] provides an asynchronous API for backup/restore the document index, through internal document index service requests.

## Backup

Send a PATCH `BackupRequest` request to `/core/management`.  
In the request body, `kind` and `destination` are required.

`BackupRequest`:
- `kind`:
  specify `com:vmware:xenon:services:common:ServiceHostManagementService:BackupRequest`
- `destination`:
  URI that accept PUT request from target xenon host.
  The PUT request contains a zip file that contains files used by document index service.


Sample:

```shell
> curl -X PATCH -H "Content-Type: application/json" -d '
{
  "kind": "com:vmware:xenon:services:common:ServiceHostManagementService:BackupRequest",
  "destination": "http://localhost:8000/myservice"
}
' http://localhost:8000/core/management
```

In this example command, _myservice_ will receive a PUT request which contains a zip file.  
e.g.: see [`MinimalFileStore.java`](https://github.com/vmware/xenon/blob/master/xenon-common/src/test/java/com/vmware/xenon/services/common/MinimalTestService.java)


## Restore

Send a PATCH `RestoreRequest` request to `/core/management`.  
**NOTE:** you **MUST** restart the service host that processed the restore request.

`RestoreRequest`:
- `kind`:
  specify `com:vmware:xenon:services:common:ServiceHostManagementService:RestoreRequest`
- `destination`:
  URI that accept GET request from target xenon host, and return a zip file that contains files used by document index service.

```shell
> curl -X PATCH -H "Content-Type: application/json" -d '
{
  "kind": "com:vmware:xenon:services:common:ServiceHostManagementService:RestoreRequest",
  "destination": "http://localhost:8000/myservice"
}
' http://localhost:8000/core/management
```

In this example command, _myservice_ will receive a GET request and expected to return a zip file.

### Time Snapshot Boundary

When `timeSnapshotBoundaryMicros` parameter is specified in restore request, recovered data will not have documents updated after the speficied time. This will make point in time recovery available from the backup.
