## with `Operation#nestCompletion()`

`nestCompletion()` can control existing *completion-handler(s)* to be invoked or not.


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




## with `OperationJoin`

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



## with `OperationSequence`

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

