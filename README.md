# system-design-concepts
_An educational repository documenting system design studies_


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
