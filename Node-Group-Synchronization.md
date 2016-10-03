# Overview
Synchronization in Xenon is the process that establishes the latest state and 
ownership of replicated service instances in a node-group and replicates this 
state to all relevant peer nodes. 

Synchronization kicks-in when a node joins or leaves a node-group. Synchronization
only applies to Stateful services that use the REPLICATION ServiceOption. Xenon 
handles synchronization for both PERSISTED and NON-PERSISTED (in-memory) services. 

This page covers the process and internals of Synchronization in-detail. It will 
be useful for readers to refresh themselves with the following **prerequisites**:

 * [Glossary](./Glossary)
 * [Programming Model](./Programming-Model)
 * [Multi-Node Tutorial](./Multi-Node-Tutorial)

# Synchronization Process

Because the joining node may not be a new node but may already contain state for the same services, the process of synchronization is responsible for establishing the best and latest state for each service instance 

![](https://lh3.googleusercontent.com/cTK6xlycgnYXqfdIqEC6UmW8xtpPjbbY7PLKyOCDsj0KO4_m039Ah80GFkChzYB19sztw0F-FRvXO303HtBoSq_OYO01UG1pq5EVb0qVBhDlqN0_hc87WspcsL8L830OQ_ZP9b_0KQQjQF2Q6ceDcdShX66DPZ_qVFNnmUGTTB0TT3V2ql7izF4rOHzzygK4d3ghtaQ5Ba3ks66BghThYRetXIueVzRqaYwQRxPtpUpdyKwLJ-jf53hA3L85PTsNWu30zAaNho4puOxAJJ-LdbqHBp9NVVj6I0l9xJLIGnSXp5RfiwOlWDIfBKszrdC8CzwGGkFufngGZMoGE_93_jA_wpMftyL8ibkewizNoUIlN70Bn-FRk6zA-KNLoI4jGon3WdjV7n9VXliZ3SrwDc_4dtyidp27D_vCGZ7iqHpfjfMwrhL_t7TObWdE2MMD5EqYKSOzVy-IoS1V_LqZwszOgybNbDndrezjbv8p84Bg7l5xlxFMuHveYgvLUfy5EW-ZFrHyBDYhJ3u50BKp4ECV7OQejR7vxgjDd5nfl75YieeNp5iC7xN5IxNFU3oDICSi5f4M0ZBNY6HJTKcIs1GZ7KSpTZqT5tgyDKemcgf-8anA=w2548-h1390-no)

