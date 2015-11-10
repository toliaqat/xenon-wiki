# Overview
> **Note: THIS PAGE DESCRIBES A WORK-IN-PROGRESS**

This page discusses our plan for building a test suite to test the coming **transactions** feature in DCP.

# Services/Documents
I suggest the following services/documents to be used by the tests:
- Bank: Name
- Person: Id, Name
- Account: Id, Name, Balance, BankLink, OwnerLink

We will add fields as needed.  
An account belongs to exactly one person and one bank, while a bank may have multiple accounts and so does a person.

# Gradual increasing complexity
We will start with some basic tests and add complexity incrementally:
- Basic CRUD:
   - Open transaction Tr, create A accounts, commit Tr
   - Query count of accounts and verify it is A
   - Open transaction Tx, for each account set balance to 100, commit Tx
   - Query accounts and verify each one has a balance of 100
   - Open transaction Tz, delete all accounts, commit Tz
   - Query accounts and verify count is 0
- Transaction Flow: operation with transaction after it's been completed
- Visibility within transaction
   - Open transaction Tr
   - For i:1..A:
      - Create account Ai
      - Query and verify number of accounts is i
      - Set Ai balance to 100
      - Read account Ai and verify balance is 100
   - Cancel Tr
   - Query accounts and verify count is 0
- Short transactions
   - For i:1..A:
      - Open Transaction Ti and create account Ai
      - If i is odd cancel Ti, otherwise set Ai balance to 100 and commit Ti
   - Query accounts and verify count == A/2 and total balance == 100*A/2
   - Delete all accounts and verify count == 0
- TBD: a mix of transactional and non-transactional operations on the same services (including deletion)
- TBD: expiring services during a transaction
- Single client, multiple active transactions
   - For i:1..A:
      - Open transaction Ti, create account Ai
      - If i is odd set Ai balance to 100 (under transaction Ti)
   - For i:1..A:
      - For j:1..i
         - Read account Ai under transaction Tj
         - Verify: if i is odd and j==i then balance==100, otherwise account Ai is not found
   - For i:1..A: commit Ti
   - Query accounts and verify count == A and total balance == 100*A/2
   - Delete all accounts, query accounts and verify count == 0
- Multi-document transactions
   - Open transaction, create A accounts, set balance of each to 100, commit
   - For k:1..30:
      - Select random i != j and i,j between 1 and A
      - Select random amount less than or equal to 3
      - Open transaction, transfer amount from Ai to Aj, if k is odd abort, otherwise commit
   - Query accounts and verify total balance == 100*A
   - Delete all accounts, query accounts and verify count == 0
- Multi-Document, multiple active transactions
   - Open transaction, create A accounts, set balance of each to 100, commit
   - For k:1..30:
      - Select random i != j and i,j between 1 and A
      - Select random amount less than or equal to 3
      - Open transaction Tk, transfer amount from Ai to Aj
   - For k:1..30: if k divides with 5 abort Tk, otherwise commit Tk
   - Query accounts and verify total balance == 100*A
   - Delete all accounts, query accounts and verify count == 0
- Multi-Document, concurrent transactions
   - Open transaction, create A accounts, set balance of each to 100, commit
   - For k:1..30: do in background:
      - Select random i != j and i,j between 1 and A
      - Select random amount less than or equal to 3
      - Open transaction Tk, transfer amount from Ai to Aj
   - Join-wait until all transactions perform transfer
   - For k:1..30: if k divides with 5 abort Tk, otherwise commit Tk
   - Query accounts and verify total balance == 100*A
   - Delete all accounts, query accounts and verify count == 0
- Atomic Visibility non-transactional
   - Same as "multi-document, concurrent transactions" but run an "observer" client in parallel:
      - Query accounts and verify total balance == 100 *A
      - Repeat until number of accounts != 0
- Atomic Visibility transactional
   - Same as "multi-document, concurrent transactions" but run an "observer" client in parallel:
      - Open transaction T
      - Query accounts and verify total balance == 100 *A
      - Commit T
      - Repeat until number of accounts != 0
- TBD: Multi-Document-Type, concurrent transactions (will use the relationships among documents)
- TBD: Partial failures (will induce host failures to test recovery), including service owner failure, coordinator failure
