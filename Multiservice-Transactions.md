> This is a work in progress -- feedback is welcome!

## Overview

In  this page,  we  describe the  design of  transactions  for Xenon.  The
motivation behind transactions  is to provide a means  for developers to
group a series of operations to (potentially different instances of) Xenon
services. Operations in such scenario are  part of a larger context, and
enable A and I in ACID for groups of operations in Xenon (as Durability is
handled directly by Xenon based on user preferences).

* Atomicity: the whole transaction will either succeed or fail -- it
  won't be half applied with half updates/inserts/deletions succeed and
  half fail.

* Isolation:  transactions  accessing   services  concurrently  do  not
  interfere with each other.

## Requirements

Similar to other Xenon features requirements drive the protocol choice and implementation.

 * Support for all Xenon Actions - Transactions include both updates and reads (POST, PUT, PATCH, DELETE, GET)
 * Decentralized - Transactions should evolve independent of each other unless a service
is involved in multiple concurrent transactions
 * Atomic - The side effects of the operations within the transaction become visible only after the transaction
is succesfull, in a atomic way
 * Isolated - Side effects should not be visible unless the transaction commits. In Xenon terms the updates
are not visible to other transactions or clients, and notifications are not issued while the transaction is
in progress
 * Service side only - The client can drive a transaction without using a client library or
complex interaction patterns. The protocol should require a single additional operation: a "commit"
which is issued by the client when all reads and updates that are part of the transaction have been issued
 * Ability to do a 2PC validation before commit - A common requirement for service transactions is
to run a validation stage, that has visibility into the transaction operations, *after* the client has
issued a commit but before it receives the commit response. This allows for referential integrity on
complex data models

We have two transaction protocols:
* [Optimistic](https://docs.google.com/document/d/1A4PE97CosbTcEFb4xyKs-bsI8EkyYzvdgzN5Ft8ahxY/edit#heading=h.7vr5rrx28pab)
* [Simple](https://github.com/vmware/xenon/wiki/simple-transactions)
