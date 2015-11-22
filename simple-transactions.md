# Overview
The Simple Transaction Service is a simple, lightweight transaction coordinator for Xenon services/documents that supports the ACID properties.

# Usage
To perform transactional operations using the Simple Transaction Service, a client typically:
* Issues a POST to the Simple Transaction Factory Service with a body that optionally contains a transaction id (txid) UUID in the POST body's documentSelfLink (otherwise a random UUID will be allocated automatically)
* Sets the transactionId property on each operation it wants to invoke in the context of the transaction by calling Operation.setTransactionId(txid).
* Sends the operation
* If all operations within the context of the transaction succeed, commits the transaction by sending an EndTransaction commit request to the transaction coordinator; otherwise, sends an EndTransaction abort request to the coordinator.

The TestSimpleTransactionService class contains a number of unit tests that demonstrate that usage.