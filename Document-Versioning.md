Each document in Xenon has `documentVersion` field that gets incremented on every update to that document. Using this field and along with `documentOwner` and `documentEpoch` Xenon resolves conflicts during synchronization between multiple nodes.

## Changing version retention limit
By default Xenon keeps 1000 old document versions. You can change that setting by implementing `getDcoumentTemplate()` method in your service to tune the disk requirements of your service. Following is the example showing how to set the version retention limit in your service.

    @Override
    public ServiceDocument getDocumentTemplate() {
        ServiceDocument template = super.getDocumentTemplate();

        // instruct the index to only keep the most recent N versions
        template.documentDescription.versionRetentionLimit = ThisService.VERSION_RETENTION_LIMIT;
        return template;
    }

## Querying a particular version

Whenever we do `GET` on a service, latest version of that service document is returned by Xenon. We can create a query to get a particular version. Following is an example code that creates a QueryTask for retrieving a document with particular version. It includes any deleted versions as well by setting `QueryOption.INCLUDE_DELETED`.

    public static QueryTask buildVersionQueryTask(long targetVersion, String documentSelfLink) {
        QueryTask.Query selfLinkClause = new QueryTask.Query()
                .setTermPropertyName(ServiceDocument.FIELD_NAME_SELF_LINK)
                .setTermMatchValue(documentSelfLink)
                .setTermMatchType(QueryTask.QueryTerm.MatchType.TERM);

        QueryTask.QuerySpecification querySpecification = new QueryTask.QuerySpecification();
        querySpecification.resultLimit = 10;
        querySpecification.options = EnumSet.of(
                QueryTask.QuerySpecification.QueryOption.INCLUDE_ALL_VERSIONS,
                QueryTask.QuerySpecification.QueryOption.INCLUDE_DELETED,
                QueryTask.QuerySpecification.QueryOption.TOP_RESULTS,
                QueryTask.QuerySpecification.QueryOption.EXPAND_CONTENT);

        QueryTask.NumericRange<?> versionRange =
                QueryTask.NumericRange.createEqualRange((Long)targetVersion);

        versionRange.precisionStep = Integer.MAX_VALUE;
        QueryTask.Query versionClause = new QueryTask.Query()
                .setTermPropertyName(ServiceDocument.FIELD_NAME_VERSION)
                .setNumericRange(versionRange);

        querySpecification.query.addBooleanClause(selfLinkClause);
        querySpecification.query.addBooleanClause(versionClause);

        return QueryTask.create(querySpecification).setDirect(true);
    }

