# About

This document provides guidelines for authoring core xenon code. It is meant for developers contributing features, fixes to xenon-common where great scrutiny must be applied in order to avoid functional, performance and design regressions.

# Guidelines

## Be careful with locks

- Never call a method with holding a lock
  - e.g. call `op.complete` within synchronized block
- Do not allow locks in service code

## Challenge the use of ServiceOption.CONCURRENT_UPDATE_HANDLING

In both core xenon services or application services built on top of xenon, question the use of this option. It is meant for advanced in memory services only and carries great risk, since it likely requires custom locking and state handling.

## Scrutinize every change in the fast path (a.k.a I/O path)

Fast path is the code path that gets executed by all service to service xenon operations such as handling post/get, replication, etc.  
Be extremely cautious when adding new lines of code in the fast path because:

- Fast path is always and frequently executed
- Adding one method or a few instructions in the I/O path slows everything (even if its 10ns per op, it adds up when doing 1M ops / sec / node)
- Always review code and prevent:
  1. Additional and unnecessery memory allocations
  1. Locks
  1. Logging - Avoid logging, even at fine level, at all costs

## Run perf tests when touching the I/O PATH

Whenever modifying core xenon code, establish a baseline before the change, and after the change, on an isolated, multi core system with no other processes running. Run the tests below at least 5 times to get a median value and supply a large enough updateCount / ServiceCount arguments to the tests.
Following tests are part of performance validation:

1. `TestLuceneDocumentIndexService.throughputXxx()`
1. `TestStatefulService.throughputInMemoryXxx()`
1. `TestNodeGroupService.replication(updateCount=33, serviceCount=3000)`

_(You have to modify some variables to perform high load use case. Default number is set to very low)_

See [Service Framework Performance](service-framework-performance) for how to run the performance test suite.

## Document all new features and core services

Add at least a new wiki page including:
- Overview
- Simple URI
- Simple REST URI
- etc.

See NodeGroupService, QueryTaskService wiki pages for examples
