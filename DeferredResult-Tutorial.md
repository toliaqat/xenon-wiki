Due to the asynchronous nature of Xenon, it’s often necessary to coordinate and chain multiple operations in complex workflows. Relying entirely on completion handlers, in these cases in particular, can lead to unmanageable and hard to trace code (aka callback hell).

We expose a set of methods in `ServiceRequestSender` (implemented in `ServiceHost`, `Service`, etc) – `sendWithDeferredResult`, which return an instance of `DeferredResult`. For those familiar with [`CompletableFuture`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html), `DeferredResult` implements identical interface, suitable for monadic style of chaining code blocks. In fact the implementation of `DeferredResult` encapsulates `CompletableFuture`.

In this tutorial we’ll go over common patterns of using `sendWithDeferredResult` and `DeferredResult` showcasing the most commonly used constructs. For detailed explanation of all the constructs refer to the [`CompletableFuture`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html) and [`CompletionStage`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionStage.html) documentations and the many tutorials available online.

# Basic usage pattern:
Compare the following code snippets:

With `DeferredResult`
```java
@Override
public void handleGet(Operation get) {
    Operation operation = Operation.createGet(...);
    this.sendWithDeferredResult(operation, ExpectedType.class)  // returns DeferredResult<ExpectedType>
            .thenApply(this::processResult)
            .thenAccept(response -> get.setBody(response))
            .whenCompleteNotify(get);
}

private Response processResult(ExpectedType result) {
    // business logic
}
```

Without `DeferredResult`

```java
@Override
public void handleGet(Operation get) {
    Operation operation = Operation.createGet(…);
    operation.setCompletionHandler((o, e) -> this.processResult(o, e, get));
    this.send(operation);
}

private Response processResult(Operation o, Throwable f, Operation original) {
    if (f != null) {
        original.fail(f);
        return;
    }
    ExpectedResult result = o.getBody(ExpectedType.class);
    // business logic
    original.setBody(response).complete();
}
```

Few things to note here:

1. The code flow when using `DeferredResult` is more natural – we send the request, then define how we are processing the result and then we notify the "main" get operation.
2. Notifying the get operation in case of normal code execution or error/exception is done in one place. `whenCompleteNotify` is equivalent to `try ... finally` code block.
3. One less `Operation` (`o`) object to deal with when using `sendWithDeferredResult` :)

The usage of this pattern is demonstrated in pretty much all of the samples that work with `DeferredResult` – `SamplePreviousEchoService::handleGet`; `LocalWordCountingSampleService::handleGet`, etc

# Running operations is sequence:
Chaining different operations is achieved using `thenCompose()`. The function passed to `thenCompose` is called with the result of the previous stage, if completed normally, and is expected to return `DeferredResult`. The code flow is switched to follow the completion of the returned `DeferredResult`.  

```java
@Override
public void handleGet(Operation get) {
    this.sendWithDeferredResult(Operation.create..., X.class)
             .thenCompose(this::transform)
             .thenAccept(result -> get.setBody(result))
             .whenCompleteNotify(get);
}

private DeferredResult<Y> transform(X intermediate) {
    // some logic
    return this.sendWithDeferredResult(Operation.create..., Y.class);
}
```

For a working example you can check `LocalWordCountingSampleService::countWordsInLocalDocuments`:

```java
private DeferredResult<WordCountsResponse> countWordsInLocalDocuments() {
    return fetchLocalDocumentLinks() // this is asynch call to retrieve all the child services
            // when done, we call processDocuments which results in another asynch operation
            .thenCompose(this::processDocuments)
            // when the operation(s) scheduled by processDocuments are done we aggregate the responses
            .thenApply(WordCountingSampleService::aggregateResponses);
}
```

# Recovering from errors:
Sometimes we want to do some additional processing in case of error, for example return a default value. This can be achieved by using the methods `handle` or `exceptionally`.

```java
private static final String DEFAULT_VALUE = "Xenon";

// --- snip ---
Operation op = ...
this.sendWithDeferredResult(op, X.class)
        .exceptionally(this::recoverFromError)
        .thenAccept(...)
        .whenCompleteNotify(...);
// --- snip ---

private X recoverFromError(Throwable f) {
    // Due to failures in the different stages of the execution, the failures might get wrapped
    // in CompletionException, we need to extract the original
    if (f instanceof CompletionException) {
        f = f.getCause();
    }
    // examine the original exception
    if (f instanceof ServiceNotFoundException) {
        return DEFAULT_VALUE;
    }
    // .exceptionally accepts a Function, so we need to either return a value (recover from the error)
    // or throw an Exception. In any case the compiler will remind us!
    // We were unable to recover, re-throw the original, wrapped in CompletionException (which is RTE).
    throw new CompletionException(f); 
}
```

The function passed to exceptionally is called only if there was an error in the previous stages. `handle` on the other hand expects a BiFunction and similar to the `whenComplete*` set of methods is always called with the result of the previous stages and the error if any. The difference with `whenComplete*` is that it alters the propagated result.
Here is a snippet from `LocalWordCountingSampleService` demonstrating the pattern:

```java
private DeferredResult<WordCountsResponse> processDocument(String documentLink) {
    return this.fetchDocument(documentLink) // this is asynch call
            .thenApply(this::countWords)
            .thenApply(WordCountsResponse::fromWordCounts)
            // If any of the previous stages fail – the async call or the transformations after it,
            // the function in the exceptionally stage will be called.
            .exceptionally(error -> {
                // in case of an error recover by returning empty map
                this.logWarning("Failure while processing %s, excluding from result!",
                        documentLink);
                return WordCountsResponse.fromFailure();
            });
}
```

Building on the previous example, suppose we want to recover from the error by sending request to another service. One approach we can take here is to use `handle()` and provide `BiFunction` which returns `DeferredResult`. And at the next stage switch the `DeferredResult` by using `thenCompose`:

```java
// --- snip ---
Operation op = ...
this.sendWithDeferredResult(op, X.class)
        .handle(this::recoverFromError)
        .thenCompose(Function.identity()) // x -> x
        .thenAccept(...)
        .whenCompleteNotify(...);
// --- snip ---

private DeferredResult<X> recoverFromError(X result, Throwable f) {
    if (f == null) { // no error
        // return an already completed DeferredResult!
        return DeferredResult.completed(result);
    }
    // in case of error, we can unwrap the exception here, check it, etc
    return this.sendWithDeferredResult(..., X.class);
}
```

# Running multiple operations in parallel:
Orchestrating the execution of multiple operations in parallel can be achieved with `DeferredResult.allOf()`. 

```java
List<URI> uris = ...;
List<DeferredResult<X>> deferredResults = uris.stream()
        .map(uri -> this.sendWithDeferredResult(Operation.createGet(uri), X.class))
        .collect(Collectors.toList());
DeferredResult<List<X>> finalResult = DeferredResult.allOf(deferredResults);
finalResult
        .thenApply(...) // Process the result, which is List<X>
        .whenComplete(...); // finally
```

The method `allOf(List<DeferredResult<T>)`, missing in `CompletableFuture`, transforms a `List<DeferredResult<T>>` into a `DeferredResult<List<T>>`.  

Code from `WordCountingSampleService` that demonstrates the pattern:

```java
private DeferredResult<List<WordCountsResponse>> processDocuments(List<String> documentLinks) {
    if (documentLinks == null || documentLinks.isEmpty()) {
        return DeferredResult.completed(Collections.emptyList());
    }
    // Fan-out and process the individual documents in parallel
    List<DeferredResult<WordCountsResponse>> deferredResults = documentLinks.stream()
            .map(this::processDocument)
            .collect(Collectors.toList());
    return DeferredResult.allOf(deferredResults);
}
```

# Others

## Logging the failure when running multiple operations in parallel:

```java
List<DeferredResult<X>> deferredResults = uris.stream()
        .map(uri -> this.sendWithDeferredResult(Operation.createGet(uri), X.class)
                    .whenComplete((result, f) -> {
                        if (f != null) {
                            this.logSevere("Problem with %s", uri, f);
                        }
                    }))
        .collect(Collectors.toList());
DeferredResult<List<X>> finalResult = DeferredResult.allOf(deferredResults);
```

## Obtaining partial results in case of failure when running multiple operations in parallel:

```java
// Pair is an object that holds a pair of objects, i.e. AbstractMap.SimpleEntry
List<DeferredResult<Pair<X, Throwable>>> deferredResults = uris.stream()
        .map(uri -> this.sendWithDeferredResult(Operation.createGet(uri), X.class))
        .map(deferred -> deferred
                .thenApply(x -> new Pair<>(x, (Throwable) null))
                .exceptionally(f -> new Pair<>((X) null, f)))
        .collect(Collectors.toList());
DeferredResult<List<Pair<X, Throwable>>> finalResult = DeferredResult.allOf(deferredResults);
```

## Branching out:

```java
private DeferredResult<Y> computeValue(X x) {
    // First check if the result is cached
    if (this.cache.containsKey(x)) {
        return DeferredResult.completed(this.cache.get(x));
    }
    // Compute the value, potentially asynchronously and update the cache
    return this.sendWithDeferredResult(Operation.create..., Y.class)
            .whenComplete((y, f) -> {
                if (f == null) { // no failures
                    this.cache.put(x, y);
                }
            });
}

// --- snip ---
    Operation op = ...
    this.sendWithDeferredResult(op, X.class)
            .thenCompose(this::computeValue)
            .whenCompleteNotify(...);
// --- snip ---

