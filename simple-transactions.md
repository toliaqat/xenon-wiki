# Overview
The Simple Transaction Service is a simple, lightweight transaction coordinator for Xenon services/documents that supports the ACID properties.

# Usage
To perform transactional operations using the Simple Transaction Service, a client typically:
* Issues a POST to the Simple Transaction Factory Service with a body that optionally contains a transaction id (txid) UUID in the POST body's documentSelfLink (otherwise a random UUID will be allocated automatically)
* Sets the transactionId property on each operation it wants to invoke in the context of the transaction by calling Operation.setTransactionId(txid).
* Sends the operation
* If all operations within the context of the transaction succeed, commits the transaction by sending a Commit request to the transaction coordinator; otherwise, sends an Abort request to the coordinator.

The TestSimpleTransactionService class contains a number of unit tests that demonstrate that usage:
    private String newTransaction() throws Throwable {
        String txid = UUID.randomUUID().toString();

        // this section is required until IDEMPOTENT_POST is used
        this.host.testStart(1);
        SimpleTransactionServiceState initialState = new SimpleTransactionServiceState();
        initialState.documentSelfLink = txid;
        Operation post = Operation
                .createPost(getTransactionFactoryUri())
                .setBody(initialState).setCompletion((o, e) -> {
                    if (e != null) {
                        this.host.failIteration(e);
                        return;
                    }
                    this.host.completeIteration();
                });
        this.host.send(post);
        this.host.testWait();

        return txid;
    }

    private void createAccount(String transactionId, String accountId, boolean independentTest)
            throws Throwable {
        if (independentTest) {
            this.host.testStart(1);
        }
        BankAccountServiceState initialState = new BankAccountServiceState();
        initialState.documentSelfLink = accountId;
        Operation post = Operation
                .createPost(getAccountFactoryUri())
                .setBody(initialState).setCompletion((o, e) -> {
                    if (operationFailed(o, e)) {
                        this.host.failIteration(e);
                        return;
                    }
                    this.host.completeIteration();
                });
        if (transactionId != null) {
            post.setTransactionId(transactionId);
        }
        this.host.send(post);
        if (independentTest) {
            this.host.testWait();
        }
    }

    private void commit(String transactionId) throws Throwable {
        this.host.testStart(1);
        Operation patch = SimpleTransactionService.TxUtils.buildCommitRequest(this.host,
                transactionId);
        patch.setCompletion((o, e) -> {
            if (operationFailed(o, e)) {
                this.host.failIteration(e);
                return;
            }
            this.host.completeIteration();
        });
        this.host.send(patch);
        this.host.testWait();
    }
