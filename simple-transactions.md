# Overview
The Simple Transaction Service is a simple, lightweight transaction coordinator for Xenon services/documents that supports the ACID properties. It is one of the two transaction protocols supported in Xenon and is 100% external to the DCP core runtime as it is implemented as an OperationProcessingChain filter (see "Implementation" below).

# Usage
To perform transactional operations using the Simple Transaction Service, a client typically:
* Issues a POST to the Simple Transaction Factory Service with a body that optionally contains a transaction id (txid) UUID in the POST body's documentSelfLink (otherwise a random UUID will be allocated automatically)
* Sets the transactionId property on each operation it wants to invoke in the context of the transaction by calling Operation.setTransactionId(txid).
* Sends the operation
* If all operations within the context of the transaction succeed, commits the transaction by sending a Commit request to the transaction coordinator; if some operation failed (e.g. an IllegalStateException due to a transactional conflict), sends an Abort request to the coordinator. 

The TestSimpleTransactionService class contains a number of unit tests that demonstrate that usage:
The host sets the core transaction service to null and starts the Simple Transaction Factory. 
A service participating in the Simple Transaction Service protocol must inject the Simple Transaction Service Filter to its request I/O pipeline (**both in its factory and service classes**).
    
    @Before
    public void setUp() throws Exception {
        try {
            this.host.setTransactionService(null);
            if (this.host.getServiceStage(SimpleTransactionFactoryService.SELF_LINK) == null) {
                this.host.startServiceAndWait(SimpleTransactionFactoryService.class,
                        SimpleTransactionFactoryService.SELF_LINK);
            }
        } catch (Throwable e) {
            throw new RuntimeException(e);
        }
    }

    public static class BankAccountFactoryService extends FactoryService {

        public static final String SELF_LINK = ServiceUriPaths.SAMPLES + "/bank-accounts";

        public BankAccountFactoryService() {
            super(BankAccountService.BankAccountServiceState.class);
        }

        @Override
        public Service createServiceInstance() throws Throwable {
            return new BankAccountService();
        }

        @Override
        public OperationProcessingChain getOperationProcessingChain() {
            if (super.getOperationProcessingChain() != null) {
                return super.getOperationProcessingChain();
            }

            OperationProcessingChain opProcessingChain = new OperationProcessingChain(this);
            opProcessingChain.add(new TransactionalRequestFilter(this));
            setOperationProcessingChain(opProcessingChain);
            return opProcessingChain;
        }
    }

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

# Conflicts
Operations invoked on services participating in a transaction can conflict. The Simple Transaction Service conflict detection and handling is simple: if a service is enrolled in a transaction, operations outside the context of the transaction (including both transactional operations in the context of a different transaction and non-transactional operations) fail with an IllegalStateException.

In the future we might provide finer control over the isolation level, for example: the ability to perform non-transactional read on a service enrolled in a transaction.

# Queries
Queries currently do not take into account transactions automatically, but the client can use the transactionId field in queries (e.g. to include only documents that participate in a specific transaction, or to include only documents that participate in no transaction). Therefore, queries do not raise conflicts, however performing a GET on a document returned by a query raises an exception in case of a transactional conflict.

# Implementation
The implementation is captured in SimpleTransactionService, which has two parts:
* A service which acts as a transaction coordinator for a given transaction id.
* An OperationProcessingChain filter, which is injected into each service participating in the transaction.

The filter intercepts each transactional operation and enrolls the service with the transaction coordinator. It is also responsible for detecting and handling transactional operations' conflicts. The coordinator maintains the set of enrolled services. At the end of the transaction, the coordinator sends a ClearTransactionRequest to the enrolled services, which is handled by the filter.
