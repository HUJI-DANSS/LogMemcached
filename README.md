# LogMemcached

A source code of the paper LogMemcached - An RDMA based Continuous Cache Replication, that was released as part of the KBNets '17 Proceedings of the Workshop on Kernel-Bypass Networks in SIGCOMM.

The original paper can be found here: http://dl.acm.org/citation.cfm?id=3098584

An extended version of the paper can be found here: TBA.

The paper authors are: Samyon Ristov from The Hebrew University of Jerusalem, Yaron Weinsberg from Microsoft, Danny Dolev from The Hebrew University of Jerusalem and Tal Anker from Mellanox Technologies.

The code's purpose is described in the abstract below. To understand it further it's advised to read the original paper or the extended version, linked above. The paper and especially its extended version describe the system's flow, design, data structures, pseudocode, evaluation and performance analysis. This readme does not focus on these aspects. Instead, it's focuses on the technical and research aspects that are missing in this code and work.

## Abstract

One of the advantages of cloud computing is its ability to quickly scale out services to meet demand. A common technique to mitigate the increasing load in these services is to deploy a cache.

Although it seems natural that the caching layer would also deal with availability and fault tolerance, these issues are nevertheless often ignored, as the cache has only recently begun to be considered a critical system component. A cache may evict items at any moment, and so a failing cache node can simply be treated as if the set of items stored on that node have already been evicted. However, setting up a cache instance is a time-consuming operation that could inadvertently affect the service’s operation.

This work addresses this limitation by introducing cache replication at the server side by expanding Memcached (which currently provides availability via client side replication). This work presents the design and implementation of LogMemcached, a modification of Memcached’s internal data structures to include state replication via RDMA to provide an increased system availability, improved failure resilience and enhanced load balancing capabilities without compromising performance and with introducing a very low CPU load, while keeping the main principles of Memcached’s Design Philosophy.

##  Environment

### Hardware

Mellanox Technologies MT27520 Family. ConnectX-3 Pro.

### Linux

We ran under the code with Ubuntu 14.10, kernel 3.16.0-43 with OFED version 3.18. It has not been tested under any other OS, although there should not be any problems with most of the other Debian distributions.

## Known issues, bugs and missing features

* prepend and append command's items does not replicate well - the items are replicated fine, but they are allocated twice. First, there is an allocation in "process_update_command" in memcached.c, see "FIXME" comment near "item_alloc". Then, they are alloceted again in "do_store_item" in memcached.c. The allocated item in process_update_command will stay "ITEM_DIRTY" forever, crushing the replication process. Without replication, the system will work fine (one prepend/append item will stay "ITEM_DIRTY" and the other one will be stored normally).
* during replication, the systems waits for "ITEM_DIRTY" flag to disappear - it make sense, but what if because of some problem, an item stayed in "ITEM_DIRTY" state forever? Like in the bug above? This will cause the replication process to crush. The system should wait some time until the "ITEM_DIRTY" flag is removed, and if it's not removed, to keep further if possible, or to ping the master that some problem occurred - causing the master to mark the item with "ITEM_CORRUPTED" flag. Beware of consistency problems here.
* memory ghost - sometimes, during replications, weird things are happening. One such place is "do_store_replication" in memcached.c, see "FIXME" comment there. safe_buf (and buf) sometimes, every hundreds of thousands of reads is modified during the run. For example, there is an 'if' statement (if ((flags & ITEM_STORED) && !(flags & ITEM_DIRTY) && !(flags & ITEM_CYCLE))) that shows that it->it_data_flags is not ITEM_STORED, but then, during the run, in the last 'else' statement in that 'if' statement, it->it_data_flags is ITEM_STORED - hence, someone modified it. As a patch for now, the flag to the stack before doing any comparison (i.e. there is no direct comparison to safe_buf (or buf). Probably the problem is caused because the NIC, or some CPU reordering. Not sure. The bug is reproduces easily if the comparison is done to it->it_data_flags directly (i.e. to safe_buf or buf), and there is a client that store big items (32KB or above) nonstop.
* memory barriers - the code uses plenty memory barriers (asm volatile("": : :"memory")) - they are placed before critical places, but there is no proof that they really works as expected, and that they all in the right places. Need to recheck each one of them.
* sleep for synchronization - in backup_rdma.c a usleep(1000) statement appears for a few times, which is clearly slow down the system. The reason they are there is that without them the replication just fails, especially when it occurs fast enough. To reproduce the problem, remove the usleep(1000) statements, start a LogMemcached master and a slave, and start a client (like memaslap) that will set (without get) items to the master. The replication will fail right away.
* binary API - currently, LogMemcached supports only the ASCII API, while Memcached supports a binary API too, that behaves a bit differently than the ASCII API.
* multiple slaves - currently, Logmemcached support only one replication slave, but it should support at least 3 slaves, and it's better to support any arbitrary number of slaves (of course, there is a performance costs to replicate via multiple slaves).
* read only mode - replication slaves should support a read only mode, in which, while they perform replication (i.e. while they act as slaves), they should allow incoming "get" commands, and forbid all others. Currently, there is no mechanism that forbids other commands. Be careful with "get" commands too. In LRU or LFU modes a get command can "issue" a "set" command. See more details in the paper.
* TBA
## Extending the research

* TBA
