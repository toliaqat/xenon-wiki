# Overview
Synchronization kicks in when a node joins or leaves the xenon node-group.
Synchronization establishes the latest state and ownership of services
in a node-group and replicates the state to all relevant peer nodes.

Synchronization only applies to Stateful services that use the REPLICATION
ServiceOption. Xenon supports synchronization for both PERSISTED and
NON-PERSISTED (in-memory) services.

This page covers the process and internals of Synchronization in-detail.
It will be useful for readers to refresh themselves with the following
**prerequisites**:

 * [Glossary](./Glossary)
 * [Programming Model](./Programming-Model)
 * [Multi-Node Tutorial](./Multi-Node-Tutorial)

# Synchronization Process
Synchronization gets kicked-off when a node joins or leaves the xenon node
group (Fig-a below). Technically speaking, synchronization is triggered
whenever a node-group converges after going through a change i.e. node(s)
getting added or removed. The process of node-group convergence is described
in more detail here (todo).

Each node in the node-group runs synchronization and goes through all
factory services to determine the synchronization OWNERs for that factory
(Fig-b below). Ownership here is determined by consistent-hashing through the
factory self-link. After each node has established independently the factory
services it owns, it tries to establish consensus by contacting other nodes in the
node-group to ensure that everybody else also considers this node the
Synchronization owner for the specific factory Service. This is done to avoid
scenarios where two nodes may be running synchronization for the same Factory
service. If however, two nodes conflict each other on ownership, this
usually indicates that the node-group is going through changes and a later
node-group convergence event will resolve the conflict and re-trigger
synchronization.

![](https://lh3.googleusercontent.com/57QxEv8lYuYb6ab3ztE_qhdIE9wN-Ufkj9OALu4oOhSKh0z1PnxowcNvXuSWg6YEWYm0xPFHC3s58uTW8JIYzcTFBdAXjHZ0IyhJuXprwVdWruFZq6i5CT_X6hJDQuKK9QO44vNPglQOAt5G-Io4cc6QdQbnpHR6t-evjXNYQP9nqmw5geSrJ8C12FAVKOtWDp_89xqUmEYH3DeurgfyQKjFINdiXk96MoMNYczgTgmtMnHdRX3IzSFrJH3Qra1493VmObnCXdy4nRHafqB2zZSp5J2bmcEueEYIs0UsAbSWg902iUmqT5KdjW4eVB0o8xqsqvhR2wRW6IOv9L7nOZ-cv9ypagt95vgYK9hOTDEmtM32VNsyVCnaTBnPbtaNZl5CREyRG2PjtwFeeLa1tJJ6W7v8ht58pOCLtHvIyxZL-YgEnIlgFubawU1OzBa-F6hwr9gEyWkysIazZtBeYs27KQ3WltA5f7rySVVvTrWvn3Z6FtKo2JuWjP8o-H40TDJbvZtLuAMA16CDwJtBXtZ3hkiQtiw8_AteKcqkh4gzi0teRtLiDHO2LL8gcU8nOcwJR2PjMWcB2b9Ek6J2CmY1gjpwpW2eZnpNm9rcE-DlpRtX=w2048-h910-no)

Once ownership has been established, each node starts synchronizing child
services for the factories it owns. Figure C below indicates
that Node-C has been selected as the Synchronization owner for Examples
Service i.e. /core/examples.

To synchronize the child services, each node kicks off an instance of the
SynchronizationTaskService per factory. The Synchronization Task starts by
making a broadcast query to compute the union of all documentSelfLinks for
the child services of that factory (Fig-d below). The broadcast query ensures
that each child service known to any node in the node-group gets synchronized
to its latest state.

![](https://lh3.googleusercontent.com/lLozKpDUeY2n_pCWUImHAW_VDTwjanR7tFbQrOndLAoFghSit6VSzHB2iFnq_lktu-rW3WlcHBkmTEFDoYOGAcwTaz_bxaPiiMU6psTzPBXNfzJZF1Z7Q4Ja0s6iyO0RmtkCHbWh2dDCbu7aHapvgggG-QGzNZijNnWi2l2IJY2vn_UU80iddX1sxIgTaWnQ-7Dltcr7G9Z6cbBLjxKBNzkaybuNEjpQNe3QjZs9jSTIHYIcat4F2rC9KTnfU9lTSUjD4Zl1vWq23aWFJOwWE-Clc-8IzJVXyEIrgzpAvHbsxG42xQT33AaVi44morolpbpf3-QiyUdyhAkedZ2uBLblLnjGlwOVu23WGh8Xz159y-zln4QsAFKef0nKgVQlHv0Xvb34BgVTEi-Oiy2sHAhR0N2KDv2dldW1J-35D5ZfouZbJWe_HL1Rujd84Wu5M_upr9pvYR_TbGio6W_V-W6RNXvm6g-dgiCJ9tbIGnhcwitQzfuWyESo7AhXRE4UAz2FgY8H6TO9LOCjbvpRblzvPJkYfwSQDCSw4VIlDcvL02Y27qiFw8rtLWGcYmvcTOBvb64FCkkkdlvMZNYNQ7vsuV_qeysY0-n-hrKCYTMj0ktw=w2048-h916-no)

After computing the child service documentSelfLinks for each a given factory,
the synchronization task starts making SYNCH-POST requests to owner nodes of
these child services. SYNCH-POST is a HTTP POST request tagged with a specific
pragma to indicate to the child service owner node that it needs to synchronize
this child service. Figure-e indicates that Node "C" sent SYNCH-POST requests
to A and E for child1 and child2.

The requests are forwarded to the owner nodes to ensure that the synchronization
request goes through the same Operation processing pipeline that is used to
process any other active updates to the child service. This increases reliability
and reduces chances of conflict because of different nodes processing updates
for the same service.

Also, notice that even though Node "E" just joined the node-group and will most
likely not have any state for the child2, still receives the SYNCH-POST request.
This is because child service synchronization always considers the latest state
of the child service by querying each node in the node-group. This will become 
more clear in the coming discussion.

Once the child service owner has received the SYNCH-POST request, it starts
processing it by making a broadcast query to all peer nodes to get the latest
version of the Sevice (Fig-f).

![](https://lh3.googleusercontent.com/w-KCQPuiQf-JxWgOrKUO0PoRUurVagAveLd9990UdcJ8t4sAD4iGhPw5SFPbML6jNEjGAYLu8gD6zqRQO6uEVUb-NKd0o8E_qwdkRb5f4-AUppVawXOB9UAulewMk9ppoF2-hHWMur3r1ELoatc2SOUG1mdTtnEpOAmYBWhK1PAy_9M5vePv1mCmH8fYjjPwBBY0bmCyz-Sbk-d9CO0eDT4nQKKml0sGkw2DNppsNHlrNyn8wbGwG-O9bDQSdv5E9e2Enmqs2iUdcCucGkdrzdakIhJVXtLbT0Cuz6FZqhAaHTHKWyMUFM2VX6Z4F3ID7Hd5kXUbMAKu4ZLOYW_gwDwFMIOXkVnWFZc2UCq8YrK5qrcAHfnNigbYpdfqN7vYHFAWX8YGm6yyscTirFmgSKAo58-_AVOXv1zisBUmTWicenMnpBHHuomYgNnfCs5vgbW8QVQTtphHL81AKEeKNG3vxdndDA_Ge_MVsGSEkIMFZVviAnjkSbwpl4fqu3MbtpP35jBNrLZhWeCLIfPtsHM8CQa0H9IzTmYyJtrWnjkxB9g18oVGHyu_m-mkc8JaVhBiXzqzz4RRmb2ZZWLt0SHKxxTP293dKRBLmOI6Dy4svsrK=w2048-h945-no)

After the child service owner has received the latest state from it's peers,
it will go over the results to compute the latest state. The process of computing
latest state is critical for Xenon to recover service state reliably in the
face of errors and network partitions. To do this, Xenon uses documentVersion
and documentEpoch per Service Document. The documentVersion field in a monotonically
increasing number indicating the number of updates the local service instance has 
processed since creation. The documentEpoch field is also a monotonically 
increasing number that gets incremented each time a child service owner changes. 
Given the above details, the algorithm of compute best state is as follows:
 * Select the document(s) with the highest documentEpoch.
 * If there are more than one documents with the highest epoch, select the 
   document(s) with the highest documentVersion.
 * If there are more than one document with the highest epoch and highest 
   documentVerison, then xenon picks one of the documents randomly. Although,
   we could still have scenarios because of Network splits that two different
   documents may exist with the same epoch and document version, Xenon currently
   does not solve this problem.

Once owner node has computed the latest state of the child service, it will 
broadcast the new state to all peer nodes in the node-group (Fig-g). Also, if the
latest state is different than the local state of the owner node, it will update
it's state. 

After each child-service has finished synchronization, the SynchronizationTaskService
for will complete execution and mark the Factory service's stats as AVAILABLE (Fig-h).

![](https://lh3.googleusercontent.com/Zm2TZjiE2P_Kif1JprLAC0f0j_6R0o3QqsOvaISXle92OvibfxHEsqrh2jL270mLqPXx72kmgSCEH4PZ7TyE4-Uk3XSMmx0gyC4_GBZ0xnhEcxAIbi_c87_iyVgJA5Wtzlt6gaGi_6hIE1-HO4V1ws584E9zK1XwjiKfb12kcMoyueEov8sAGaGbBtGQQf3mPrmY_818jkWPtUStCUEOAabK6B-sO268jqbya-kDliFJdpAkgncR5907t3a0IJSHfr8AHD6SHWsU2ITdARnA5LoVSVxceCv6kY-xj2Hf-HjmlvPQaUmHbKT2kqG3uuaKYLaPfzHVJu5qGndfZYssyKLP9v_hinBAN0Y8SRDUxmlcRQu7F_LymAslmR4QG6jLNaNm-Chr-DUUtF7OsLo1CLywZoYygXdLnogbKqrffEVMkXHTYqgMuu2fxYw7ptKNLlZpIEh6AktDIE9vnL0xz-vJI2aZNKcDq3dZAZINCPLNlBNz17Y9v6AuPLKeaJvE-RjqFJ0xUd9trAsGGTKC2cgj5lEp8HKkJXV8bOyP09L8eipOcBcbLHDamyGGUmqBXDhRr2AKSmbnjY0JBoJ__JMNH9W5lnhDKMZYRtSJz0rWlPNc=w2048-h956-no)

The above approach for Synchronization provides the following benefits:
 * Synchronization work-load gets uniformly distributed across all nodes in the 
   node-group. This increases concurrency and reduces the overall amount of time
   taken for synchronization.
 * Each child service synchronization request goes through the owner nodes as 
   discussed above. This ensures that any in-flight update requests for the 
   child services get serialized without running into state conflicts.
 * The latest state of each child service gets computed and replicated to all
   peer nodes thus ensuring that the entire state of node-group becomes consistent.

# Synchronization Scenarios
 
The process of synchronization also gets invoked in scenarios other than node-group
changes. These include:

 * **Node Restarts**: Generally speaking, for replicated services a node restart 
   is no different than a node leaving and joining a Xenon node-group. However, 
   for non-replicated services, xenon still needs to make sure that all locally
   stored child services get start. To handle this scenario, Xenon uses the 
   same Synchronization process, with the caveat that the SYNCH-POST request in
   Figure d (above) always gets sent to the local node. Also the local node skips
   steps outlined in Figures f and g to avoid querying other nodes for the latest
   state and just considers the local state to start the service. 
   
 * **On-demand Synchronization**: It is possible that while the synchronization 
   process was running a child service that has not been synchronized yet, receives
   an update request (PATCH, PUT or DELETE). Instead of blocking the write request
   or failing it, Xenon detects that the local service needs to be synchronized
   at that time and kicks-off synchronization right away i.e. steps outlined by
   Figures f and g. After the service has been synchronized and the owner node 
   has the latest state, xenon replays the original write request.
   
 * **Synchronizing OnDemandLoad Services**: Because 