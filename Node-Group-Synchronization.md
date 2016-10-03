# Overview
Synchronization kicks in when a node joins or leaves the xenon node-group. 
It is the process that establishes the latest state and ownership of services
instances in a node-group and replicates this state to all relevant peer nodes. 

Synchronization only applies to Stateful services that use the REPLICATION 
ServiceOption. Xenon handles synchronization for both PERSISTED and NON-PERSISTED
(in-memory) services. 

This page covers the process and internals of Synchronization in-detail. It will 
be useful for readers to refresh themselves with the following **prerequisites**:

 * [Glossary](./Glossary)
 * [Programming Model](./Programming-Model)
 * [Multi-Node Tutorial](./Multi-Node-Tutorial)

# Problem Statement
When a **new node joins the node-group**, it is important for Xenon to replicate the
necessary state to the new node so that it can start serving user requests.  
Depending on the amount of state and number of service instances involved, this 
could be a very I/O intensive process (both disk and network). The synchronization
process should scale out the workload here (of copying the latest state to the new
node) as much as possible to reduce the amount of time it takes to bring it to a 
ready state. At the same time, correctness of state is also very important. The 
new node joining the node-group will become the OWNER for a number of existing 
service instances, so the synchronization process is also required to replicate 
the latest state to the new node. 

But what if the joining node is not a new node? What if this was an existing node
that got split because of **Networking Partitioning**, and is now joining back 
to the node-group. It is also possible that the node joining the node-group
may have more latest state than the rest of the nodes. So, the synchronization 
process is also required to make sure to update the existing peer nodes in the 
node-group if the joining node has more recent and up to date information.

_STILL WIP .... _

![](https://lh3.googleusercontent.com/cTK6xlycgnYXqfdIqEC6UmW8xtpPjbbY7PLKyOCDsj0KO4_m039Ah80GFkChzYB19sztw0F-FRvXO303HtBoSq_OYO01UG1pq5EVb0qVBhDlqN0_hc87WspcsL8L830OQ_ZP9b_0KQQjQF2Q6ceDcdShX66DPZ_qVFNnmUGTTB0TT3V2ql7izF4rOHzzygK4d3ghtaQ5Ba3ks66BghThYRetXIueVzRqaYwQRxPtpUpdyKwLJ-jf53hA3L85PTsNWu30zAaNho4puOxAJJ-LdbqHBp9NVVj6I0l9xJLIGnSXp5RfiwOlWDIfBKszrdC8CzwGGkFufngGZMoGE_93_jA_wpMftyL8ibkewizNoUIlN70Bn-FRk6zA-KNLoI4jGon3WdjV7n9VXliZ3SrwDc_4dtyidp27D_vCGZ7iqHpfjfMwrhL_t7TObWdE2MMD5EqYKSOzVy-IoS1V_LqZwszOgybNbDndrezjbv8p84Bg7l5xlxFMuHveYgvLUfy5EW-ZFrHyBDYhJ3u50BKp4ECV7OQejR7vxgjDd5nfl75YieeNp5iC7xN5IxNFU3oDICSi5f4M0ZBNY6HJTKcIs1GZ7KSpTZqT5tgyDKemcgf-8anA=w2548-h1390-no)

