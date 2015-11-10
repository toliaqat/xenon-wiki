# Custom Request Routing

In order to facilitate separating the implementation of logical REST operations to different methods, and to enable discovery of such operations (from a client perspective), a construct called RequestRouter has been introduced.

A service developer can create a RequestRouter and register with it pairs of (matcher, handler), also known as "route". When an incoming request arrives at a service, the I/O pipeline checks if a RequestRouter has been registered with the service, and if so - it checks if it contains one or more handlers of the corresponding Action (get, put, patch, etc.). If it does, the handling of the request is delegated to the RequestRouter, which invokes the handler of the first matcher that matches the request.

RequestRouter provides built-in facilities for matching a request based on a request body field and based on a URI query param.

Here is an example of registering a "deposit" and "withdraw" handlers, taken from the BankAccountService sample. In this example incoming PATCH requests are delegated to either the handlePatchForDeposit() or handlePatchForWithdraw() methods, based on the value of the "kind" field in the request body.

    @Override
    public OperationProcessingChain getOperationProcessingChain() {
        if (super.getOperationProcessingChain() != null) {
            return super.getOperationProcessingChain();
        }

        RequestRouter myRouter = new RequestRouter();
        myRouter.register(
                Action.PATCH,
                new RequestRouter.RequestBodyMatcher<BankAccountServiceRequest>(
                        BankAccountServiceRequest.class, "kind",
                        BankAccountServiceRequest.Kind.DEPOSIT),
                this::handlePatchForDeposit, "Deposit");
        myRouter.register(
                Action.PATCH,
                new RequestRouter.RequestBodyMatcher<BankAccountServiceRequest>(
                        BankAccountServiceRequest.class, "kind",
                        BankAccountServiceRequest.Kind.WITHDRAW),
                this::handlePatchForWithdraw, "Withdraw");
        OperationProcessingChain opProcessingChain = new OperationProcessingChain();
        opProcessingChain.add(myRouter);
        setOperationProcessingChain(opProcessingChain);
        return opProcessingChain;
    }
    
    void handlePatchForDeposit(Operation patch) {
        BankAccountServiceState currentState = getState(patch);
        BankAccountServiceRequest body = patch.getBody(BankAccountServiceRequest.class);

        currentState.balance += body.amount;

        setState(patch, currentState);
        patch.setBody(currentState);
        patch.complete();
    }

    void handlePatchForWithdraw(Operation patch) {
        BankAccountServiceState currentState = getState(patch);
        BankAccountServiceRequest body = patch.getBody(BankAccountServiceRequest.class);

        if (body.amount > currentState.balance) {
            patch.fail(new IllegalArgumentException("Not enough funds to withdraw"));
            return;
        }
        currentState.balance -= body.amount;

        setState(patch, currentState);
        patch.setBody(currentState);
        patch.complete();
    }

Another benefit of using a RequestRouter to register handlers is that you can specify a description text with each route. This allows a client to discover the logical operations with their description using the "/template" utility suffix. Here's an example output:

    {
       "balance": 0.0,
       "documentDescription": {
       "propertyDescriptions": {
       ...
       },
       "serviceCapabilities": [
       ...
       ],
       "serviceRequestRoutes": {
         "PATCH": [
         {
          "action": "PATCH",
          "condition": "body.kind\u003dDEPOSIT",
          "description": "Deposit"
         },
         {
          "action": "PATCH",
          "condition": "body.kind\u003dWITHDRAW",
          "description": "Withdraw"
         }
        ]
      },
      ...
     },
    ...
    }