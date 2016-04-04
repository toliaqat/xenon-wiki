# About

This document provides guidelines for writing code in xenon.


# Guidelines

## Be careful with locks

- Never call a method with holding a lock
  - e.g. call `op.complete` within synchronized block
- Do not allow locks in service code

Always ask question if you see `CONCURRENT_UPDATE_HANDLING` options.


## Be careful in the fast path (a.k.a I/O path)

Fast path is a code path that always get executed by the xenon operations such as handling post/get, replication, etc.  
Extra cautious to add new lines of code in the fast path because:

- Fast path is always and frequently get executed
- Adding one method in I/O path slows everything (even in 10ms)
- Always perform checks for:
  1. Memory allocation
  1. Locks
  1. Logging  <== _BAD (never do logging in fast path)_
	  - Try not to use `logFine(...)`. It still assign memory.

## Run perf tests when touching the I/O PATH

Following tests is a performance test suit:

1. `TestLuceneDocumentIndexService.throughputXxx()`
1. `TestStatefulService.throughputInMemoryXxx()`
1. `TestNodeGroupService.replication(updateCount=33, serviceCount=3000)`

_(You have to modify some variables to perform high load usecase. Default number is set to very low)_

See [Service Framework Performance](service-framework-performance) for how to run the performance test suite.


## Write wiki page per service

New core service (or component) in xenon, you need to have wiki page including:
- Overview
- Simple URI
- Simple REST URI
- etc.
