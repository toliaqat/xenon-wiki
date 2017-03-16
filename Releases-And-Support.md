# Releases

Xenon is under active development, and new versions are released regularly. Releases can be found on the GitHub [releases page](https://github.com/vmware/xenon/releases), and artifacts can be found on [Maven Central](https://repo1.maven.org/maven2/com/vmware/xenon/) and [Sonatype](https://oss.sonatype.org/content/groups/public/com/vmware/xenon/). Release numbers follow [Semantic Versioning][semver]. For more information on the release process, see the [Cutting a Release](Cutting-a-release) page.

Since the framework is still fairly early in its lifecycle, releases come at a fast cadence -- we aim to release once every two weeks, with specific point releases sometimes more frequent than that. Each release is subject to the same set of test and reliability requirements imposed by the Xenon [CI environment](Developer-Guide#ci--cd). For more information on the test requirements for each release, see the [Testing](Developer-Guide#testing) section of the developer guide.

[semver]: http://semver.org/

## Long-Term Support

The following releases are supported by the development team, including the creation of new critical releases as issues are identified, for a period of one year following initial release.

* Xenon 1.1 -- the latest release is [v1.1.0-CR3](https://github.com/vmware/xenon/releases/tag/v1.1.0-CR3-release)

## Releases

### v1.4.1

Xenon 1.4.1 contains minor changes and bug fixes.

For a full description of the changes in this release, please consult the [changelog](https://github.com/vmware/xenon/blob/master/CHANGELOG.md#141).


### v1.3.7

Xenon 1.3.7 contains new features, performance enhancements, and bug fixes.

For a full description of the changes in this release, please consult the [changelog](https://github.com/vmware/xenon/blob/master/CHANGELOG.md#137).

#### Major Changes

* **Purge all document versions for recreation** -- When deleted self link is recreated with POST + PRAGMA_FORCE_INDEX_UPDATE, the index service will purge all previous documents. This avoids duplicate versions, for the same self link appearing in the index, which can happen due to synchronization or migration (even if the runtime does do a best effort to increment the version when a self is recreated)

* **Support for QueryOption#TIME_SNAPSHOT** -- The new query option will return results that contain latest versions of documents as on a given time. QuerySpecification#timeSnapshotBoundaryMicros will allow specifying the time.

* **Remove LuceneBlobIndexService** -- The service was originally used for binary serializing service context in pause / resume, which now uses a custom file based service (since 1.2.0)


### v1.3.6

Xenon 1.3.6 contains new features, performance enhancements, and bug fixes.

For a full description of the changes in this release, please consult the [changelog](https://github.com/vmware/xenon/blob/master/CHANGELOG.md#136).

#### Major Changes

* **Support for Operation Tracing in Xenon UI** -- Introduce "Operation Tracing" feature to Xenon UI that allows users to trace operations sent or received by a service host instance via an interactive query builder and examine results visually.

* **GroupBy query fixes for Numeric Fields** -- Fix groupBy on numeric fields. When annotated with PropertyIndexingOption.SORT, add a SortedDocValuesField for the numeric property. The change has no impact on how query specification is written. However a blue-green update is necessary in order for previously indexed documents to be queried using groupBy on a numeric field. Documents which match the query but have the groupBy term missing are returned under a special group "DocumentsWithoutResults".

* **Index upgrade support for Xenon v1.1.1 and older** -- Added support for index upgrade from pre 1.1.1 version. Should be used only as a last resort. If the xenon.kryo.handleBuiltInCollections system property is set to false, index contents can be read back ONLY from pre-1.1.1 created indeces. If documents don't hold instances created by Collections.emptyList() and friends upgrade will still be possible without using this property.

### v1.3.5

Xenon 1.3.5 contains new features, performance enhancements, and bug fixes.

For a full description of the changes in this release, please consult the [changelog](https://github.com/vmware/xenon/blob/master/CHANGELOG.md#135).

#### Major Changes

* **Support for HTTP/2 over TLS** -- this release supports HTTP/2 connection sharing to secure endpoints. Making use of this feature requires that client projects make use of OpenSSL for ALPN support through inclusion of a Netty tcnative library in the classpath; without this library, the system will fall back to HTTP 1.1 for connections to secure endpoints. For details, see the [wiki page](Netty-Pipeline#http2-and-tls-with-netty).

* **Gateway service** —- this release adds a native gateway service which can be used to facilitate blue/green upgrades of Xenon node groups. The gateway service can be used during upgrades to pause incoming traffic while data is being migrated to the new node group. Once data has been migrated, the gateway can be resumed to point all incoming traffic to the new node group.

* **Version lookup caching** -- this release implements version lookup caching inside the document index service. This is a significant optimization for general queries which return many of results over documents with multiple versions. This change is transparent to clients and does not change the document query API.

### v1.3.4

Xenon 1.3.4 contains new features, performance enhancements, and bug fixes. Notably, this change adds an implicit result limit for queries.

For a full description of the changes in this release, please consult the [changelog](https://github.com/vmware/xenon/blob/master/CHANGELOG.md#134).

#### Major Changes

* **Implicit Query Result Limits** -- this release adds an implicit result limit to Lucene index queries which are not paginated and do not specify an explicit result limit. The default value is 10,000 results. Well-behaved clients should not be impacted, but queries which return more results than the implicit limit will be **failed** by the framework. For more details, see the [tracker issue](https://www.pivotaltracker.com/story/show/130467457).

### v1.3.3

Xenon 1.3.3 contains bug fixes.

For a full description of the changes in this release, please consult the [changelog](https://github.com/vmware/xenon/blob/master/CHANGELOG.md#133).

### v1.3.2

Xenon 1.3.2 contains performance enhancements and bug fixes.

For a full description of the changes in this release, please consult the [changelog](https://github.com/vmware/xenon/blob/master/CHANGELOG.md#132).

### v1.3.1

Xenon 1.3.1 contains new features, performance enhancements, and bug fixes. Notably, this change adds support for delegating authentication to an external service.

For a full description of the changes in this release, please consult the [changelog](https://github.com/vmware/xenon/blob/master/CHANGELOG.md#131).

#### Major Changes

* **Delegated authentication** -- this release adds support for redirection of un-authenticated API requests to an external service for authentication, as well as verification of access tokens generated by such a service. For full documentation, see the [external authentication](External-Authentication) page.

* **Sort by multiple fields** -- this release adds support for queries to specify multiple fields to use when sorting results. For full documentation, see the [query task service](QueryTaskService#sorting-results) page.

#### Breaking Changes

* **Removal of OperationOption.SEND_WITH_CALLBACK** -- SEND_WITH_CALLBACK was an experimental protocol which allowed the re-use of an HTTP connection for multiple requests. This functionality has largely been supplanted by HTTP/2 in Xenon with the caveat that HTTP/2 does not currently support TLS; this is forthcoming in a subsequent release.

### Previous Releases

For releases prior to v1.3.1, please consult the [changelog](https://github.com/vmware/xenon/blob/master/CHANGELOG.md).
