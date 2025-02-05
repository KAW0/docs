﻿# Change Vector
---

{NOTE: }

* Change vectors are the RavenDB implementation of the [Vector clock](https://en.wikipedia.org/wiki/Vector_clock) concept.
  They give us partial order over modifications of documents in a RavenDB cluster.

* Change vectors are updated every time a document is [modified](../../../server/clustering/replication/change-vector#concurrency-control--change-vectors) 
  in any way.

* In this page:
   * [What are Change Vectors Constructed From?](../../../server/clustering/replication/change-vector#what-are-change-vectors-constructed-from)
   * [How are Change Vectors used to Determine Order?](../../../server/clustering/replication/change-vector#how-are-change-vectors-used-to-determine-order)
   * [Concurrency Control & Change Vectors](../../../server/clustering/replication/change-vector#concurrency-control--change-vectors)
   * [Change Vector Comparisons](../../../server/clustering/replication/change-vector#change-vector-comparisons)
   * [Concurrency Control at the Cluster](../../../server/clustering/replication/change-vector#concurrency-control-at-the-cluster)
   * [Database Global Change Vector](../../../server/clustering/replication/change-vector#database-global-change-vector)
   * [After Restoring a Database from Backup](../../../server/clustering/replication/change-vector#after-restoring-a-database-from-backup)

{NOTE/}

---

{PANEL: What are Change Vectors Constructed From?}

A change vector is constructed from entries, one per database. 
.
It looks like this:  
`[A:1-0tIXNUeUckSe73dUR6rjrA, B:7-kSXfVRAkKEmffZpyfkd+Zw]`

This change vector is constructed from two entries,  

`A:1-0tIXNUeUckSe73dUR6rjrA` and `B:7-kSXfVRAkKEmffZpyfkd+Zw`.  

Each entry has the following structure `Node tag`:`ETag`-`Database ID`, so `A:1-0tIXNUeUckSe73dUR6rjrA` means that
the document was modified on node `A` its local `ETag` is `1` and the database ID is `0tIXNUeUckSe73dUR6rjrA`. Entries accumulate 
as the number of database instances in the database group grows. If an instance is no longer being used, its entry can be removed 
from the change vector to save space using the [UpdateUnusedDatabasesOperation](../../../client-api/operations/maintenance/clean-change-vector).  

An `ETag` is a value that is scoped to a database instance on a particular node and is guaranteed to always increase. It is used internally for many purposes and is the natural
sort order for many operations (indexing, replication, ETL, etc).
The database ID is used for cases where the node tag is not unique, which could happen when using external replication or restoring from backup.

{PANEL/} 

{PANEL: How are Change Vectors used to Determine Order?}

Given two change vectors `X` and `Y` we would say that `X` >= `Y` if foreach entry of `X` the value of the ETag is greater or equal to the corresponding entry in `Y` and `Y` has no entries that `X` doesn't have.  
The same goes for <= we would say that `X` <= `Y` if foreach entry of `X` the value of the ETag is smaller or equal to the corresponding entry in `Y` and `X` has no entries that `Y` doesn't have.  
We would say that `X` <> `Y` (no order or conflict) if `X` has an entry with a higher ETag value than `Y`, and `Y` has a different entry where its ETag value is greater than X.

{PANEL/}

{PANEL: Concurrency Control & Change Vectors}

RavenDB defines some simple rules to determine how to handle concurrent operations on the same document across the cluster. 
It uses the document's change vector.
This allows RavenDB to detect concurrency issues when writing to a document and throws an exception when the document 
has been concurrently modified on different nodes during network partition to prevent data corruption.

Every document in RavenDB has a corresponding change vector. 
This change vector is updated by RavenDB every time the document is changed. 
This happens when on document creation and 
any modification such as `PUT`, `PATCH`, `DELETE` or their bulk versions. 
A delete operation will also cause RavenDB to update the document change vector, 
however, at that point, the change vector will belong to
the document tombstone (since the document itself has already been deleted).

A change vector is present in the document's metadata and each time
a document is updated, the server will update the change vector. 
This is mostly used internally inside RavenDB for many purposes 
(conflict detection, deciding what documents a particular
subscription has already seen, what was sent to an ETL destination, etc) 
but can also be very useful for clients.

In particular, the change vector is _guaranteed_ to change whenever the document changes and can be used as part of optimistic concurrency checks. A document modification can all specify an expected change vector for a document (with an empty change vector signifying that the document does not exist). In such a case, all operations in the 
transaction will be aborted and no changes will be applied to any of the documents modified in the transaction.

{PANEL/}

{PANEL: Change Vector Comparisons}

Conceptually, comparing two change vectors means answering a question - which change vector refers to an earlier event.  

The comparison is defined as follows:  
  
* Two change vectors are equal when and only when all etags _equal_ between corresponding node and database IDs
* Change vector A is larger than change vector B if and only if all etags are _larger or equal_ between corresponding node and database IDs
* Change vector A conflicts with change vector B if and only if at least one of the etags 
  is _larger, equal, or has node etag (and the other doesn't)_ and at least one etags is _smaller_ between 
  corresponding node and database IDs
  
Note that the change vectors are unsortable for two reasons:

* Change vectors are unsorted collections of node tags/etag tuples, they can be sorted in multiple ways
* Conflicting change vectors cannot be compared
  
### Example 1
Let us assume two change vectors, v1 = [A:8, B:10, C:34], v2 = [A:23, B:12, C:65]  
When we compare v1 and v2, we will do three comparisons:

* A --> 8 (v1) < 23 (v2)
* B --> 10 (v1) < 12 (v2)
* C --> 34 (v1) < 65 (v2)
  
Corresponding etags in v2 are greater than the ones in v1. This means that v1 < v2

### Example 2
Let us assume two change vectors, v1 = [A:18, B:12, C:51], v2 = [A:23, B:12, C:65]  
When we compare v1 and v2, we will do three comparisons:

* A --> 18 (v1) < 23 (v2)
* B --> 12 (v1) = 12 (v2)
* C --> 51 (v1) < 65 (v2)
  
Corresponding etags in v2 are greater than the ones in v1. This means that v1 < v2


### Example 3
Let us assume two change vectors, v1 = [A:18, B:12, C:65], v2 = [A:58, B:12, C:51]  
When we compare v1 and v2, we will do three comparisons:

* A --> 18 (v1) < 58 (v2)
* B --> 12 (v1) = 12 (v2)
* C --> 65 (v1) > 51 (v2)
  
Etag 'A' in v1 is smaller than in v2, and Etag 'C' is larger in v1 than in v2. This means that v1 conflicts with v2.  

{PANEL/}

{PANEL: Concurrency Control at the Cluster}

RavenDB implements a multi master strategy for handling database writes. This means that it will _never_ reject a valid write to a document (under the assumption that if you tried to write
data to the database, you're probably interested in keeping it). This behavior can lead to certain edge cases. In particular, under network partition scenario, it is possible for two clients
to talk to two RavenDB nodes and to update a document with optimistic concurrency check. 

The concurrency check is done at the _local node_ level, to ensure that we can still process writes in the case of a network partition or partial failure scenario. That can mean that two
writes to separate servers will both succeed, even if each write specified the same original change vector, because each server did the check independently. Under such a scenario, the 
generated change vectors for the document on each server will be different, and as soon as replication between these nodes will run the databases will detect this conflict and resolve
it according to your conflict resolution strategy.

In practice, this kind of scenario is rare, since RavenDB attempts to direct all writes to the same node for each database under normal conditions to
ensure that optimistic concurrency checks will always happen on the same machine.

{PANEL/}

{PANEL: Database Global Change Vector}

The database global change vector is the same entity as the one used by the data it contains.  
The value of the _global change vector_ entries is determined by the maximum value of all the change vectors contained in its data.  

E.g, a database containing the following documents:  

* Document `A` with _change vector_ `[A:1-0tIXNUeUckSe73dUR6rjrA, B:7-kSXfVRAkKEmffZpyfkd+Zw]`
* Document `B` with _change vector_ `[B:3-kSXfVRAkKEmffZpyfkd+Zw, C:13-ASFfVrAllEmzzZpyrtlrGq]`

Will have the following _global change vector_:

* `[A:1-0tIXNUeUckSe73dUR6rjrA, B:7-kSXfVRAkKEmffZpyfkd+Zw, C:13-ASFfVrAllEmzzZpyrtlrGq]`

The global change vector is used in the replication process to determine which data is already contained by the database.
A document that all of his entries are lower or equal from/to the _global change vector_ is considered contained.

{NOTE: }

Note that if data is considered contained by a database it doesn't mean it is present in the database, it may already be overwritten by more up-to-date data.  

{NOTE/}

{PANEL/}

{PANEL: After Restoring a Database from Backup}

* [Snapshot backups](../../../client-api/operations/maintenance/backup/backup#snapshot) save [change vector data](../../../server/ongoing-tasks/backup-overview#backup-contents).
* [Logical backups](../../../client-api/operations/maintenance/backup/backup#logical-backup) do not save change vector data.  
  Restoring a database from a logical backup will renew the document change vector's frequently [incrementing ETag](../../../server/clustering/replication/change-vector#what-are-change-vectors-constructed-from)
  to its original value.

{PANEL/}

## Related Articles

### Client API

- [How to enable optimistic concurrency](../../../client-api/session/configuration/how-to-enable-optimistic-concurrency)
- [How to get entity change vector](../../../client-api/session/how-to/get-entity-change-vector)
- [How to clean change vector](../../../client-api/operations/maintenance/clean-change-vector)

