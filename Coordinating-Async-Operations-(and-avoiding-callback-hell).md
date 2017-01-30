# Overview

The framework offers sender (client) coordination primitives that enable concurrent, multi-stage
processing, of request / responses. For the various options, see below.

## Strong typing, functional style, with `DeferredResult`
See the [tutorial](./DeferredResult-Tutorial) on using the functional pattern for
coordinating sequence of operations and chaining their results.

```java
 Operation operation = Operation.createGet(...);
    this.sendWithDeferredResult(operation, ExpectedType.class)  // returns DeferredResult<ExpectedType>
            .thenApply(this::processResult)
            .thenAccept(response -> get.setBody(response))
            .whenCompleteNotify(get);
```

## Completion stacks with `Operation#nestCompletion()`

`nestCompletion()` can control the order existing *completion-handler(s)* are invoked


```java
Operation op1 = Operation
    .createGet(...)
    .setCompletion((o,e) -> {   // let's call this "handler"
      if(e != null){
        parentGet.fail(e);
        ...
        return;
      }
      parentGet.complete();    // control parent Operation here
      ...
    });

op1.nestCompletion((o,e) -> {   // let's call this "nested-1"
  if(e != null) {
    op1.fail(e);
  }
  // business logic here
  ...
  op1.complete();    // this will invoke "handler"
}).nestCompletion((o,e) -> {   // let's call this "nested-2"
  if(e != null) {
    op1.fail(e);
  }
  // business logic here
  ...
  op1.complete();    // this will invoke "nested-1"
})
```

*Invocation Order*

- "nested-2" -> "nested-1" -> "handler"




## Concurrent processing with `OperationJoin`

```java
Operation op1 = Operation
        .createGet(...)
        .setCompletion((o, e) -> {
            if (e != null) {
                // ...
                return;
            }
            // ...
        });
Operation op2 = Operation
        .createGet(...)
        .setCompletion((o, e) -> {
            if (e != null) {
                // ...
                return;
            }
            // ...
        });


JoinedCompletionHandler jh = (ops, failures) -> {
    if(failures != null){
        parentGet.fail(...);
    }
    
    parentGet.complete();  // handle parent Operation here
};

OperationJoin.create(op1, op2).setCompletion(jh).sendWith(this);
```

*Invocation Order*

- op1 - op2 - joined
- op2 - op1 - joined


op1 and op2 will run in parallel.
Once both completion handlers are called, then joined completion handler will be called.



## Sequencing with `Operation#appendCompletion()`

```java
Operation op = Operation.createGet(...);

// initial handler
op.setCompletion((o, e) -> {
    ...
    // calling "o.complete()" or "o.fail()" will invoke next chain
    o.complete();  // trigger next chain
});

// append-1 handler
op.appendCompletion((o, e) -> {
  if (e != null) {
    // invokes next handler with passed exception
    // o.fail(e);

    // invokes next handler with NEW exception
    // o.fail(new RuntimeException(...));

    // invokes next handler with success
    // o.complete();

    // just calling "return" will NOT invoke next handler
    return;
  }
  ...
  o.complete();  // trigger next chain
});

// append-2 handler
op.appendCompletion((o, e) -> {
  if (e != null) {
    o.fail(e);
    return;
  }
  ...
  o.complete();
});
```

*Invocation Order*

- initial -> append-1 -> append-2  
*(Order is guaranteed)*



## Sequencing with `OperationSequence`

```java
Operation op1 = Operation
        .createGet(...)
        .setCompletion((o, e) -> {
            if (e != null) {
                // ...
                return;
            }
            // ...
        });
Operation op2 = Operation
        .createGet(...)
        .setCompletion((o, e) -> {
            if (e != null) {
                // ...
                return;
            }
            // ...
        });



JoinedCompletionHandler jh = (ops, failures) -> {
    if(failures != null) {
       // ...
       parentGet.fail(...);
    }
    parentGet.complete();   // handle parent Operation here
};

OperationSequence.create(op1).next(op2).setCompletion(jh).sendWith(this);

```

*Invocation Order*

- op1 -> op2 -> joined  
*(Order is guaranteed)*
 


*Note:*

When there is a failed operation, `OperationSequence` will continue calling next operations.

```java

// assume op2 fails
OperationSequence.create(op1).next(op2).next(op3).next(op4).setCompletion(jh).sendWith(this);
```

*Invocation Order*

- op1 -> op2 -> op3 -> op4 -> joined  
*(joined handler will receive failure object from op2)*

