# System Design Concepts
_An educational repository documenting system design studies_



## Table of Contents

- [Database Index Implementations](#database-index-implementations)
  - [Hash Index](#hash-index)
    - [Pros](#pros)
    - [Cons](#cons)
  - [LSM Tree / SSTable Index](#lsm-tree--sstable-index)
    - [Pros](#pros-1)
    - [Cons](#cons-1)
  - [B Tree Index](#b-tree-index)
    - [Pros](#pros-2)
    - [Cons](#cons-2)
- [A.C.I.D. Transactions](#acid-transactions)
  - [Atomicity](#atomicity)
  - [Consistency](#consistency)
  - [Isolation](#isolation)
  - [Durability](#durability)
- [Read Committed Isolation](#read-committed-isolation)
  - [Dirty Reads](#dirty-reads)
    - [Example](#example)
    - [How to prevent it](#how-to-prevent-it)
      - [Row Locking](#row-locking)
      - [Snapshot Isolation](#snapshot-isolation)
  - [Dirty Writes](#dirty-writes)
    - [Example](#example-1)
    - [How to prevent it](#how-to-prevent-it-1)
      - [Row Locking](#row-locking-1)
- [Snapshot Isolation](#snapshot-isolation-1)
  - [Implementation](#implementation)
- [Read Skew](#read-skew)
  - [Example](#example-2)
- [Write Skew](#write-skew)
  - [Example](#example-3)
- [Phantom Writes](#phantom-writes)
  - [Example](#example-4)
- [Materializing Conflicts](#materializing-conflicts)
- [2 Phase Locking (2PL)](#2-phase-locking)
  - [Race Conditions](#race-conditions)
  - [Predicate Locking](#predicate-locking)
  - [Index Range Locking](#index-range-locking)
- [Serializable Snapshot Isolation (SSI)](#serializable-snapshot-isolation)
  - [Example](#example-5)
- [2PL vs SSI](#2pl-vs-ssi)
- [Actual Serial Execution](#actual-serial-execution)
- [Column Oriented Storage](#column-oriented-storage)
  - [Notable Downsides](#notable-downsides)
  - [Column Compression](#column-compression)
  - [Predicate Pushdown](#predicate-pushdown)
- [Serialization Frameworks](#serialization-frameworks)
  - [Examples](#examples)

 

## Database Index Implementations

### Hash Index
Implemented entirely in-memory. Effectively a hash table, where keys are respective keys, and values are respective disc location. This allows for O(1) lookup in-memory for disc location, greatly reducing time of disc scan.

#### Pros: 
* Extremely fast -- In-memory hash map lookups are near instant

#### Cons:
* Poor performance with range queries -- each value in range needs to be individually queried in-memory and scanned on disc.
* Low memory -- because hashmap is in-memory, we are greatly restricted by how many keys we can index. Once we run out of space, there's not much we can do. Moving hashmap to disc would greatly damper performance.


### LSM Tree / SSTable Index
LSM tree is self-balancing sorted binary tree storing key and value at each node. It is effectively an in-memory write buffer. A write-ahead log is maintained to make LSM Tree resilient to server failures. Once the tree is full, it is flushed onto disc in the form of an SSTable, populated via in-order traversal of tree.

SSTable is log file of key-value pairings. When accessing a key that is not found in LSM Tree, system goes to disc to search for key in SSTables.

To reduce wasted space caused by duplicate keys in SSTables, compaction takes place in the background. This is the process of merging SSTables. When a duplicate key is found, the more-recent value is kept. This process happens in linear time [O(n)] because SSTables are sorted.

To speed up look-up times in SSTables, a sparse-index hash table is maintained. This has _some_ keys from the SSTables in sorted order, along with their respective disc memory locations. By finding a key that is organizationally less than and greater than our target key, we can perform a binary search to find where our target key should exist, if it does in the range.

Bloom filters are also used to speed up look-up times in SSTables. They help indicate whether an SSTable _may_ contain the desired key, effectively communicating whether it is a waste of resources to search through the SSTable.

#### Pros:
* Fast writes at scale -- in-memory write buffers allows for heavy-write indexing
* Range querying -- due to sorted nature of data, range queries are optimized

#### Cons:
* Reads are not optimal -- scanning through SSTables on disc is costly, and is most apparent when looking for old key or key that does not exist
* Background SSTable compaction takes up resources


### B Tree Index
This index is stored entirely on disc in the format of a tree of blocks. Each block has "n" keys, and "n-1" references. At the bottom block, rather than references, the block contains the keys' respective values. Given target key, traverse through each layer of the tree, and find where key falls in order of block's keys. Take respective reference route to next block, and so on, until key is found (if it exists).

This results in logarithmic look-up time for all keys. Reads are quite quick.

Writes are more costly since they are all on disc. In addition, each block can only have "n" keys. So, if a key belongs on a block that already has "n" keys, then the block needs to be split in half, the key added to the split block, and the parent of the blocks updated accordingly. This can result in a cascade of block splits, making writes all the more costly.

#### Pros:
* Fast reads at scale -- all reads are done in logarithmic time, and most trees only have 3-4 levels
* Range querying -- due to sorted nature, range querying is quite efficient

#### Cons:
* Writes are inefficient -- all writes happen on disc and can cause need for rebalancing of tree which is expensive



## A.C.I.D. Transactions
Good to strive for because they provide high data reliability and integrity. Most of this can be achieved using a write-ahead log, but Isolation is a bit more tricky. It is common to be a bit more relaxed in systems when striving for Isolation.

### Atomicity
All-or-nothing. Either the entirety of a database transaction will succeed, or nothing at all.

### Consistency
When failures occur, they do so gracefully. Failures should not result in invalid data, or broken data invariants.

### Isolation
Transactions should appear to occur independently of one another. Essentially, prevent race conditions and dead locks.

### Durability
Committed writes should never be lost.



## Read Committed Isolation
This level of isolation is acheived by protecting against Dirty Reads and Dirty Writes.

### Dirty Reads
This occurs when uncommited values are read.

#### Example
Say you have 2 transactions, T1 and T2. T1 updates `accountBalance` from $90 to $100. Immediately after, T2 reads `accountBalance` to be $100 and returns it to the server. T1 has yet to commit the update to `accountBalance` and fails, resulting in `accountBalance` being rolled back to $90. T2 did a dirty read since it read the uncommitted value of $100.

#### How to prevent it

##### Row Locking
If T1 had locked the row associated to `accountBalance`, it would have prevented all other transactions from using the row until T1 was complete, whether that be by failure or success and all writes were committed. This would slow down the system because only one transaction could use a row at a time.

##### Snapshot Isolation
An alternative to row locking is called snapshot isolation, essentially keeping a history of values when doing updates. This would have allowed T2 to read the old value of $90 since $100 was not yet committed, preventing the dirty read altogether.

### Dirty Writes
This occurs when uncommitted values are overwritten.

#### Example
Say you have 2 transactions, T1 and T2. T1's instructions are to update `favoriteColor=blue` and `favoriteFood=pizza`. T2's instructions are to update `favoriteColor=green` and `favoriteFood=cake`. These transactions run concurrently and are executed in the following order:
1. T1 sets `favoriteColor=blue`
2. T2 sets `favoriteColor=green`
3. T2 sets `favoriteFood=cake`
4. T1 sets `favoriteFood=pizza`
5. T1 and T2 are committed

This race condition would result in the end values being `favoriteFood=pizza` (T1) and `favoirteColor=green` (T2), which are not valid. We should have either ended with T1's values, or T2's values, not a combination of the 2.

#### How to prevent it

##### Row Locking
If T1 had locked the rows associated to `favoriteFood` and `favoirteColor`, it would have prevented all other transactions from using the row until T1 was complete, whether that be by failure or success and all writes were committed. Effectively, T1 would have executed, committed, then T2 would have executed, committed, and the final result would be all of T2's values being present. This would slow down the system because only one transaction could use a row at a time, but the race conditions would have been prevented.



## Snapshot Isolation

This is effectively the process of storing old database values to prevent race conditions.

### Implementation

A monotonic stack is used to incrementally assign Transaction IDs to transactions.

Whenever a value is written to the DB, a Transaction ID is associated to it to indicate the age of the value. The higher the Transaction ID the newer the value.

Old values are not overwritten, so each row can have multiple values per column.

When reading values, transactions choose the value that has the highest Transaction ID as long as it is less than the current transaction's Transaction ID. This prevents the current transaction from reading another transaction uncommitted values.

Compaction is utilized to remove old values that are no longer needed.



## Read Skew

This is a race condition that occurs when a transaction is doing a batch read, but a concurrent transaction modifies values causing the batch read to be corrupt and a data invariant is broken.

### Example

There are 2 concurrent threads, T1 and T2. T1 is doing a batch read that has a requirement that the values should always add up to a total of $100. To start, lets say `balance1=$40` and `balance2=$60`. The transactions execute in the following order:
1. T1 reads `balance1=$40`
2. T2 writes `balance1=$0`
3. T2 writes `balance2=$100`
4. T1 reads `balance2=$100`
5. T2 commits
6. T1 commits, effectively `balance1=$40` and `balance2=$100`.

T2 updated the balances to be `balance1=$0` and `balance2=$100`, which satifies the data invariant that balances should total $100.
However, because this update happened while T1 was reading the balances, it appears on T1 as if the balances are `balance1=$40` and `balance2=$100`, totaling $140 and breaking the data invariant.

This can be prevented using snapshot isolation. If T1 had read only read values with a Transaction ID <= itself, the uncommitted values of T2 would not have been read, and the data invariant would not have been broken.



## Write Skew

This is a race condition that occurs when 2 concurrent transactions update fields that independently would be find, but because they happen concurrently they break a data invariant.

### Example

There are 2 concurrent threads, T1 and T2. There is a data invariant stating that 1 manager must be on the clock at all times. At the start, `manager1=active` and `manager2=active`. T1 is trying to clock-out manager1, and T2 is trying to clock-out manager2. The transactions execute in the following order:
1. T1 checks if there is at least 1 other manager clocked in before clocking out for manager1.
2. T2 checks if there is at least 1 other manager clocked in before clocking out for manager2.
3. T1 writes `manager1=inactive`, effectively clocking out for manager1.
4. T2 writes `manager2=inactive`, effectively clocking out for manager2.
5. T2 commits
6. T1 commits

Due to the race condition, both manager1 and manager2 are now both clocked out, and there are no managers clocked in! The data invariant that 1 manager must be clocked in at all times is broken.

This can be prevented with row locking. If T1 had locked all managers, then T2 would not have been able to check if another manager was clocked-in until after T1 had finished clocking out. This would have prevented T2 from clocking out, and the data invariant would not have been broken.



## Phantom Writes

This is a race condition that occurs when 2 concurrent transactions create rows that independently would be find, but because they happen concurrently they break a data invariant.

### Example

There are 2 concurrent threads, T1 and T2. There is a data invariant stating that, at most, only 1 manager user can ever exist. At the start, there are no manager users. T1 is trying to create a manager user, and T2 is trying to create a manager user. The transactions execute in the following order:
1. T1 checks if a manager user exists, and sees none.
2. T2 checks if a manager user exists, and sees none.
3. T1 creates a manager user.
4. T2 creates a manager user.
5. T1 commits.
6. T2 commits.

Due to the race condition, there are now 2 manager users, breaking the data invariant stating that, at most, only 1 manager user can ever exist!

This can be prevented by materializing conflicts. See next section for details.



## Materializing Conflicts

This is used to prevent Phantom Writes. Essentially, any rows that could cause conflicts are pre-populated in the database. This enables transactions to utilize locks when creating new rows that could result in Phantom Writes.

In the prior manager creation example, T1 would have been able to lock the users table, including the non-existent manager user, then proceed with creation once it verified that no managers already exist. This would have prevented T2 from creating a second manager user, thus preventing the broken data invariant.

## 2 Phase Locking
Also known as 2PL. This is a Pessimistic Concurrency Control protocol, meaning we will end up pessimistically locking more rows than needed to prevent race conditions.

This is implemented using 2 locks per resource:
1. A shared reader lock. This can be used by multiple transactions at once, helping optimize reads. As long as the row is not being written to, reads can occur.
2. An exclusive writer lock. This can only be used by a single transaction at a time. In order for a write to occur, all reads on the resource must be complete. This wait for potentially multiple transaction executing reads on the resource can negatively impact performance of writes.

### Race Conditions
Deadlocks can occur if concurrent transactions lock each other's resources and end up endlessly waiting for one another. The database will need to have a mechanism to identify and remedy these.

Phantom writes can occur, but can be prevented through Predicate Locking and Index Range Locking.

### Predicate Locking

Transactions provide a predicate to determine which rows to lock, essentially a filter. This could mean locking all rows that are `type=manager` and `shift=night`. By locking all rows satisifying the predicate, we can ensure that rows generated to materialize conflicts are also locked, preventing phantom writes.

Because locks are expensive, and we are locking more rows than needed, this can have a significantly negative impact on performance.

### Index Range Locking

This is essentially the same as predicate locking, except we only account for the indexed portion of the predicate. So, say only `type` was indexed in the prior example, we would end up locking all rows where `type=manager`, regardless of their `shift` value. This will result in more rows being locked, but they are locked much more quickly because the rows are indexed. Essentially, lookups are done in logarithmic time rather than linear time.

There are trade-offs between predicate locking and index range locking, and each could have its place in a given system. Neither is a clear winner.



## Serializable Snapshot Isolation
Also known as SSI. This is an Optimistic Concurrency Control protocol, meaning we will end up having race conditions that require aborting, rolling back, and retrying transactions more frequently.

This is implemented using Snapshot Isolation to keep track of when a transaction is using an old version of a value.

Whenever a transaction is using an old version of a value, ie a concurrent transaction updated and commited a value used by another transaction, the transaction using the stale value is aborted, rolled back, and retried.

### Example
Pretend there are 2 transactions running concurrently. They are executed in the following order:
1. T1 reads `favoriteColor=blue`
2. T2 updates `favoriteColor=green`
3. T2 commits
4. T1 is aborted, rolled back, and retried

Because T1 was using the stale value `blue`, it needed to abort, roll back, and retry.



## 2PL vs SSI

2PL should be used in systems where there is little overlap in transactions, since many rows are locked with this protocol.
SSI should be used in systems where there is a lot of overlap in transactions, because it is more efficient to rollback and retry transactions than it is to lock resources and have transactions waiting.

## Actual Serial Execution

Check out VoltDB for an example implementation of this.

This is accomplished by restricting your database to a single thread on a single core. Effectively only one transaction can run at a time on your database, completely removing any concerns for race conditions or deadlocks!

This comes with a huge hit to processing power since concurrency is no longer feasible. To counteract this:
* Data compression can be utilized to minimize the amount of data being passed around and stored
* Storing data in-memory can greatly speed things up, but you will have limited storage compared to disk
* Stored procedures and be utilized to optimize queries, but these are known for being a nightmare to maintain

In general, this is not a common database approach due to its processing limitations



## Column Oriented Storage

When a database stores row-data, it does so in data chunks that group column data together. So, if a row has 5 columns, data will need to be written to 5 different locations on disc. As a result, data locality is geared towards doing batch queries on a single column for multiple rows.

This is great for things like data analytics. If I wanted to determine the average age of by userbase, I'd be able to quickly fetch all of the age values since they are stored next to each other on-disc.

### Notable Downsides

There are some unique drawbacks to column oriented storage.

1. All columns need to utilize the same sort order.
Because a row's data is not stored in the same place, the sort order of the data is the only thing that would allow us to fetch an entire row of data.
2. Writes require writing to multiple places on disc.
This can be optimized using a LSM Tree with SSTables. A write buffer allows us to optimize writes by performing them in-memory, then doing batch writes to disc when flushing the tree. Flushing the tree will effectively create an SSTable for each column in the rows being batch-written.

### Column Compression

If column values are often repeated, for example a user's `age` is likely a value between 0 and 100, column compression can be used to greatly reduce the amount of data passed over network and stored in the database.

Bitmap Encoding or Run Length Encoding are commonly used to represent large sets of data.

Dictionary Compression is often used when there are repeated string values. Rather than using the string value, a binary value can be assigned to represent the string value, reducing what was 2bytes/character to simply bits.

In extreme cases, compression can enable usage of in-memory storage and CPU caching, greatly improving performance.

### Predicate Pushdown

See Apache Parquet for an example implementation of this.

This adds metrics to data chunks to optimize queries. These can includes helpful values such as max, min, average, sum, etc.

For example, say you are trying to fetch all ages that are greater than 60. This would normally involve doing a linear scan through your database, potentially 1000s of data chunks with 1000s of values in each chunk. However, if your data chunks have the max value stored on them, then you can skip any chunks where the max value is under 60, since you are only fetching values over 60! This can have a huge positive impact on performance if a majority of your users are under 60.



## Serialization Frameworks

These are used to minimize the amount of data being passed over network and stored on-disc via serialization of column names, providing significant improvements to performance

Serialization and deserialization of data when communicating with the database introduces a small performance hit, but it is almost always worth the trade-off.

As a result, column names are no longer human-readable after being serialized.

### Examples

Google Protocol Buffers and Apache Thrift are 2 popular frameworks.

They both require knowledge of the schema at compile time, so they do not deal well with schema changes on-the-fly.

The schema provides a mapping of column name to unique identifier, allowing for serialization and deserialization.

Apache Avor is another popular framework that is a bit less rigid.

Avor Schemas are provided by servers when communicating with the database and stored in an Avor Schema Table, allowing for on-the-fly schema changes.

When reading values from the database, Avor will fetch the data and utilize the provided schema to keep expected values, toss unexpected values, and auto-populated missing values with schema-provided default values.
