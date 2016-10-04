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
 * [NodeGroupService](./NodeGroupService)
 * [NodeSelectorService](./NodeSelectorService)

# Synchronization Process
Synchronization gets kicked-off when a node joins or leaves the xenon node
group (**Figure A** below). Technically speaking, synchronization is triggered
whenever a node-group converges after going through a change i.e. node(s)
getting added or removed. The process of node-group convergence is described
in more detail here (todo).

Each node in the node-group runs synchronization and goes through all
factory services to determine the synchronization OWNERs for that factory
(**Figure B** below). Ownership here is determined by consistent-hashing through the
factory self-link. After each node has established independently the factory
services it owns, it tries to establish consensus by contacting other nodes in the
node-group to ensure that everybody else also considers this node the
Synchronization owner for the specific factory Service. This is done to avoid
scenarios where two nodes may be running synchronization for the same Factory
service. If however, two nodes conflict each other on ownership, this
usually indicates that the node-group is going through flux and a later
node-group convergence event will resolve the conflict and re-trigger
synchronization.

![](https://lh3.googleusercontent.com/57QxEv8lYuYb6ab3ztE_qhdIE9wN-Ufkj9OALu4oOhSKh0z1PnxowcNvXuSWg6YEWYm0xPFHC3s58uTW8JIYzcTFBdAXjHZ0IyhJuXprwVdWruFZq6i5CT_X6hJDQuKK9QO44vNPglQOAt5G-Io4cc6QdQbnpHR6t-evjXNYQP9nqmw5geSrJ8C12FAVKOtWDp_89xqUmEYH3DeurgfyQKjFINdiXk96MoMNYczgTgmtMnHdRX3IzSFrJH3Qra1493VmObnCXdy4nRHafqB2zZSp5J2bmcEueEYIs0UsAbSWg902iUmqT5KdjW4eVB0o8xqsqvhR2wRW6IOv9L7nOZ-cv9ypagt95vgYK9hOTDEmtM32VNsyVCnaTBnPbtaNZl5CREyRG2PjtwFeeLa1tJJ6W7v8ht58pOCLtHvIyxZL-YgEnIlgFubawU1OzBa-F6hwr9gEyWkysIazZtBeYs27KQ3WltA5f7rySVVvTrWvn3Z6FtKo2JuWjP8o-H40TDJbvZtLuAMA16CDwJtBXtZ3hkiQtiw8_AteKcqkh4gzi0teRtLiDHO2LL8gcU8nOcwJR2PjMWcB2b9Ek6J2CmY1gjpwpW2eZnpNm9rcE-DlpRtX=w2048-h910-no)

Once ownership has been established, each node starts synchronizing child
services for the factories it owns. **Figure C** below indicates
that Node-C has been selected as the Synchronization owner for Examples
Service i.e. /core/examples.

To synchronize the child services, each node kicks off an instance of the
SynchronizationTaskService per factory. The Synchronization Task starts by
making a broadcast query to compute the union of all documentSelfLinks for
the child services of that factory (**Figure D** below). The broadcast query ensures
that each child service known to any node in the node-group gets synchronized
to its latest state.

![](https://lh3.googleusercontent.com/lLozKpDUeY2n_pCWUImHAW_VDTwjanR7tFbQrOndLAoFghSit6VSzHB2iFnq_lktu-rW3WlcHBkmTEFDoYOGAcwTaz_bxaPiiMU6psTzPBXNfzJZF1Z7Q4Ja0s6iyO0RmtkCHbWh2dDCbu7aHapvgggG-QGzNZijNnWi2l2IJY2vn_UU80iddX1sxIgTaWnQ-7Dltcr7G9Z6cbBLjxKBNzkaybuNEjpQNe3QjZs9jSTIHYIcat4F2rC9KTnfU9lTSUjD4Zl1vWq23aWFJOwWE-Clc-8IzJVXyEIrgzpAvHbsxG42xQT33AaVi44morolpbpf3-QiyUdyhAkedZ2uBLblLnjGlwOVu23WGh8Xz159y-zln4QsAFKef0nKgVQlHv0Xvb34BgVTEi-Oiy2sHAhR0N2KDv2dldW1J-35D5ZfouZbJWe_HL1Rujd84Wu5M_upr9pvYR_TbGio6W_V-W6RNXvm6g-dgiCJ9tbIGnhcwitQzfuWyESo7AhXRE4UAz2FgY8H6TO9LOCjbvpRblzvPJkYfwSQDCSw4VIlDcvL02Y27qiFw8rtLWGcYmvcTOBvb64FCkkkdlvMZNYNQ7vsuV_qeysY0-n-hrKCYTMj0ktw=w2048-h916-no)

After computing the child service documentSelfLinks for each a given factory,
the synchronization task starts making SYNCH-POST requests to owner nodes of
these child services. SYNCH-POST is a HTTP POST request tagged with a specific
pragma to indicate to the child service owner node that it needs to synchronize
this child service. **Figure E** indicates that Node "C" sent SYNCH-POST requests
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
version of the Service (**Figure F**). This is done through the
[NodeSelectorSynchronizationService](./NodeSelectorService)

![](https://lh3.googleusercontent.com/w-KCQPuiQf-JxWgOrKUO0PoRUurVagAveLd9990UdcJ8t4sAD4iGhPw5SFPbML6jNEjGAYLu8gD6zqRQO6uEVUb-NKd0o8E_qwdkRb5f4-AUppVawXOB9UAulewMk9ppoF2-hHWMur3r1ELoatc2SOUG1mdTtnEpOAmYBWhK1PAy_9M5vePv1mCmH8fYjjPwBBY0bmCyz-Sbk-d9CO0eDT4nQKKml0sGkw2DNppsNHlrNyn8wbGwG-O9bDQSdv5E9e2Enmqs2iUdcCucGkdrzdakIhJVXtLbT0Cuz6FZqhAaHTHKWyMUFM2VX6Z4F3ID7Hd5kXUbMAKu4ZLOYW_gwDwFMIOXkVnWFZc2UCq8YrK5qrcAHfnNigbYpdfqN7vYHFAWX8YGm6yyscTirFmgSKAo58-_AVOXv1zisBUmTWicenMnpBHHuomYgNnfCs5vgbW8QVQTtphHL81AKEeKNG3vxdndDA_Ge_MVsGSEkIMFZVviAnjkSbwpl4fqu3MbtpP35jBNrLZhWeCLIfPtsHM8CQa0H9IzTmYyJtrWnjkxB9g18oVGHyu_m-mkc8JaVhBiXzqzz4RRmb2ZZWLt0SHKxxTP293dKRBLmOI6Dy4svsrK=w2048-h945-no)

After the child service owner has received the latest state from it's peers,
it will go over the results to compute the latest state. The process of computing
latest state is critical for Xenon to recover service state reliably in the
face of errors and network partitions. To do this, Xenon uses documentVersion
and documentEpoch per Service Document. The documentVersion field in a monotonically
increasing number indicating the number of updates the local service instance has
processed since creation. The documentEpoch field is also a monotonically
increasing number that gets incremented each time a child service owner changes.
Given the above details, the **algorithm to compute best state** is as follows:

 * Select the document(s) with the highest **documentEpoch**.

 * If there are more than one documents with the highest epoch, select the
   document(s) with the highest **documentVersion**.

 * If there are more than one document with the highest epoch and highest
   documentVerison, then xenon picks one of the documents randomly. Although,
   we could still have scenarios because of Network splits that two different
   documents may exist with the same epoch and document version, Xenon currently
   does not solve this problem.

Once the owner node has computed the latest state of the child service, it will
broadcast the new state to all peer nodes in the node-group (**Figure g**). Also, if the
latest state is different than the local state of the owner node, it will update
it's state.

After each child-service has finished synchronization, the SynchronizationTaskService
for will complete execution and mark the Factory service's stats as AVAILABLE (**Figure h**).

![](https://lh3.googleusercontent.com/Zm2TZjiE2P_Kif1JprLAC0f0j_6R0o3QqsOvaISXle92OvibfxHEsqrh2jL270mLqPXx72kmgSCEH4PZ7TyE4-Uk3XSMmx0gyC4_GBZ0xnhEcxAIbi_c87_iyVgJA5Wtzlt6gaGi_6hIE1-HO4V1ws584E9zK1XwjiKfb12kcMoyueEov8sAGaGbBtGQQf3mPrmY_818jkWPtUStCUEOAabK6B-sO268jqbya-kDliFJdpAkgncR5907t3a0IJSHfr8AHD6SHWsU2ITdARnA5LoVSVxceCv6kY-xj2Hf-HjmlvPQaUmHbKT2kqG3uuaKYLaPfzHVJu5qGndfZYssyKLP9v_hinBAN0Y8SRDUxmlcRQu7F_LymAslmR4QG6jLNaNm-Chr-DUUtF7OsLo1CLywZoYygXdLnogbKqrffEVMkXHTYqgMuu2fxYw7ptKNLlZpIEh6AktDIE9vnL0xz-vJI2aZNKcDq3dZAZINCPLNlBNz17Y9v6AuPLKeaJvE-RjqFJ0xUd9trAsGGTKC2cgj5lEp8HKkJXV8bOyP09L8eipOcBcbLHDamyGGUmqBXDhRr2AKSmbnjY0JBoJ__JMNH9W5lnhDKMZYRtSJz0rWlPNc=w2048-h956-no)

The above approach for Synchronization provides the following **benefits**:

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

 * **Synchronizing OnDemandLoad Services**: OnDemandLoad Services in Xenon only
   get started as a result of user requests. If not accessed, Xenon stops these
   services to reduce the memory footprint of the Xenon host. Synchronization for
   these services is a little complicated such that these services never get
   synchronized as part of node-group changes. Instead, when an OnDemandLoad service
   is requested, at that time very similar to On-demand synchronization, Xenon
   performs synchronization for that child-service and then replays the original
   write request.

# SynchronizationTaskService

As discussed earlier, the SynchronizationTaskService is responsible for
orchestrating synchronization of all child-services for a given FactoryService.
The SynchronizationTaskService is a Stateful, in-memory service. An instance of
the task is created per FactoryService. The FactoryService itself creates this
instance when the FactoryService starts, usually during host start-up. The created
task instance represents a never expiring service.

The SynchronizationTaskService represents a long-running preemptable task.
Preemptability is important here because while the task is running, a new node-group
change event may arrive that will require either the task to get cancelled or
to get restarted. The task should be cancelled because the node is no longer the
synchronization OWNER for the Factory Service. The task will get restarted if
the node-group configuration changed but the node continues to be the owner of
the FactoryService. The below diagram describes the state-machine of the
SynchronizationTaskService.

![](https://lh3.googleusercontent.com/Tm5hFYxbePSs6X9ERcdc7rzWN6zEjwA4OfFx-ECuI-bNVDUsLqr8IgOj0GDPHppWEERxpc82CxVqrFIMTcRKftJwa7uPepMfGPJFjrZF0uU2OPbJBubZp2wq7AQfQzcxrwdNWGlAAZebCu6ZhP-bl7swAEpQyT6yvzxu3x00M05fxs1JZRrbONNRnSSYLeZI2Fc88YJtsRN9YBPBktATxy_dtaKQ6J2MJ3vLkrcmBFK28jP4qe-yi7_GY8Jkh5NxLCzU689X7dkRP2C0WRWPWDplYENTYwMGxEIHzU9krgWmmTXaXoohDb6JzqmcKI5-5nBjJ-f6T3ZhJp24AF85M5ifopgiOaGFTcFxPljjchV0xHL19Zm42C1WP23erlyOGQnX22cIj5k3K9holsjN-I5sWyGsWwFWCoMa5Mmigm7lx_l4mxuEa-c_CrHY1m1Coh06s61Dn8R7n72ob9qKRjNNAoocT7iNWxzTIwrEvOH1glako4iGU2FDK0JW9RzJY-ZnrX5_9jUdL-RsdMZ0jpzo5_CWBdkf_d_L0qqNrbPF7L7KrNpSS_DMP0QSjZQjkbNIreMhfUnh71pbgEY8PP1Umk0v2MvzVEDZhDVk3s9Ia1Li=w2048-h1784-no)

**a)** The FactoryService at start-up time creates the synchronization-task in
CREATED state.

**b)** The Synchronization-task receives a PUT request everytime a node-group
change event occurs. This will move the task from CREATED to STARTED state
and QUERY sub-stage

**c)** In the QUERY sub-stage, the Synchronization-task does a broadcast query to
all peer nodes to determine the union of documentSelfLinks for that factory
service. The query performed is paginated. If there are no results returned,
the Synchronization-task jumps to the FINISHED state.

**d)** If we do find child-services, the Synchronization-task moves to the next
sub-stage SYNCHRONIZE. In the SYNCHRONIZE sub-stage the task sends out
SYNCH-POST requests to all OWNER nodes as discussed in the previous section.

**e)** After the task has sent SYNCH-POST requests for each child-service self-link
in the current page, it checks if there are more pages to process for the
broadcast query ran in step c.

**f)** If there are no more pages to process, the task jumps to the FINISHED state.

**g)** As discussed earlier, the synchronization-task is restartable for new
node-group change events. If a new node-group change event occurs, the
synch-task moves back to STARTED-QUERY stage.

**h)** It is possible that while the task is STARTED, a new node-group change event
may occur and request synchronization. In that case, the PUT request resets
the task back to RESTART sub-stage. The next time the task self-patches
it detects that it was reset and the task ends up restarting it-self by going
to the STARTED-QUERY stage.

While the node-group converges, it is possible that multiple node-group change
events can occur, and sometimes even in the wrong order. So it is critical to
only restart the Synchronization-task if the node-group change event it just 
received is actually newer. To detect this, the Synchronization-task uses the 
**membershipUpdateTimeMicros** property that is an increasing number set through 
NodeGroupService and acts as a version number of the node-group change
event. The SynchronizationTaskService stores the membershipUpdateTimeMicros
that caused the task to trigger. Only if the incoming request is for a higher
membershipUpdateTimeMicros, the task will get reset to RESTART sub-stage.

While the synchronization-task is running on one node, a node-group change event
can occur that ends up making a different node the owner for the FactoryService.
When this happens there is a chance that the new owner and the old owner both
start running the synchronization for the same FactoryService. To avoid the
old owner to keep executing synchronization, the task re-checks ownership
every time it starts processing a new broadcast query result page. If the task
discovers that it is no longer the owner, the task cancels itself.

# Code map
The above discussion covers the synchronization process and it's design. For
developers who would like to further dig into the code, the below code-map has
been added as a guide while browsing through the code.

![](https://lh3.googleusercontent.com/SWW7SHYdyTlTewjM1AddnpAPI6v2DHO1IkbjZlfiTfZCbtQWp52PqFNxKqbgmgtcTdHfiMoJCViggcnvbYa1KKkiBUO0thZlKNp2fJ2ak5-E9NJMGmHWzujK31_1Lf3lEhoFZXqeWjgQl8gT6blCqWkfQXDwDy4PDf7rNBE9A-1k2yRtFF4_-y-9l0IHMEOKgW7bbcEMqcyDTpk4MjYtA2BczsTUpuD106ROrk1pG61AX7HPGR2_s-pwiXHCr8r0mFuBTyEP4Nm0iOdDm1sQuON5ymbwRbFc9tvQ3dhjuAVwvIBlAu3imFOXFt-nt3OfRyovCSj7XlobT6Y9rVW9VDK_VBxVcbbfrRI_EiNeFbA_5yZslRv8JXlIzzuiNZe6R6elOzinEk0kddlkYWjDtt0Pc8jOLgIfOtXcKcsCbN0ZRJ6FFLri1shjhb_DlQuiNdz-Q84sD75pOZ-W_tE1XBirGsUOQ2GnZToextE5unYMXKyaDSz-EedEAFcHb168M2u9aWPitw-H38AkElrZb-L_iehMlWLqJlwdtwMUmQw89mhAMb2UiQrKIiidyeFxhSakldKNMobyj_swdB4FekIiKzGne0dp5vGsg29kJ6sbSyi7=w3280-h1790-no)
