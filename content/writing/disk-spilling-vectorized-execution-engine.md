---
title: "Disk Spilling in a Vectorized Execution Engine"
date: 2020-06-30T00:00:00Z
draft: false
tags: ["databases", "vectorized-execution", "performance", "memory-management", "cockroachdb"]
originalPost:
  site: "Cockroach Labs"
  url: "https://www.cockroachlabs.com/blog/disk-spilling-vectorized/"
---

Late last year, we shipped v1 of our [vectorized execution engine](https://www.cockroachlabs.com/docs/stable/vectorized-execution). It enables column-based query execution and speeds up complex joins and aggregations, improving analytical capabilities in CockroachDB (which is first and foremost optimized for OLTP workloads). v1 of the engine didn’t support disk spilling, which meant it couldn’t execute [certain memory-intensive queries](https://www.cockroachlabs.com/docs/stable/vectorized-execution/#disk-spilling-operations) if there was not enough memory available. Starting in [CockroachDB v20.1](https://www.cockroachlabs.com/product/whats-new/), these queries fall back to disk (also known as “spilling” to disk).

In this post, we provide a top-down explanation of how we added disk spilling to the vectorized execution engine, starting with a description of on-disk algorithms for different types of queries, and ending with a description of the single building block that all of these algorithms use. Note that disk spilling is an existing feature of the default row-at-a-time execution engine. This post specifically covers the recent addition of disk spilling to the column-at-a-time vectorized execution engine.

To learn more about why and how we created the vectorized engine, see our [How we built a vectorized execution engine](https://www.cockroachlabs.com/blog/how-we-built-a-vectorized-execution-engine/) blog post from October 2019.

## Disk Spilling Operators

### Sorts

Let’s start by covering sorts and their memory usage. A sort operator is planned when a query with the[`ORDER BY`](https://www.cockroachlabs.com/docs/stable/order-by) keyword is issued. As input, the operator takes a set of tuples, in any order, and a list of column indices to order by. It then outputs the tuples, ordered by the list of column indices.

Note that the operator must buffer the entire input before emitting a tuple, as the tuple that sorts first could be at the end of the input. As a result, the size of the input that must be buffered can exceed the amount of memory an operator is allowed to use. This memory, also called “work” memory, is limited to 64MB in CockroachDB, by default. When the work memory limit is reached, a sort must be able to spill to disk in order to sort the input fully.

To solve this problem, we took a divide-and-conquer approach, employing an external merge-sort algorithm that uses disk memory when inputs cannot be fully buffered in memory. The algorithm is broken down into two stages: sorting and merging.

In the sorting stage, the operator buffers as much input data as it can in memory (the aforementioned “work” memory), performs a column-by-column, in-memory sort, and then writes the sorted partition to disk. This stage is repeated until there are no more tuples to process.

Once the sorting stage is over, there will be N sorted partitions on disk. These partitions are then merged to emit the sorted output.

### GRACE Hash Join

A *hash join* is a type of join algorithm that joins two input streams based on a set of equality columns. It uses a hash table to store the smaller of the streams, and then probes the table with the larger stream. 

For example, suppose a user were to issue `SELECT * FROM customers, orders WHERE orders.cust_id = customers.id` to get a result where each row contains customer data and an order they issued. During the execution of this query, the hash join operator builds an in-memory hash table of the `customers` table (it is the smaller one), where the key is the customer ID. It then performs lookups using the orders table, and emits the results. 

The memory usage in this example grows as the `customers` table grows because the entire table needs to be stored in-memory. In order to respect the 64MB limit on work memory, hash joins also use a divide-and-conquer approach when spilling to disk. This type of hash join is known as a [GRACE](https://en.wikipedia.org/wiki/Hash_join#Grace_hash_join) hash join.

In a GRACE hash join, all tuples in both the `orders` and `customers` tables can be assigned to one of N on-disk partitions by hashing each tuple based on the customer ID. Because of this, all order and customer tuples with the same customer ID will end up in the same partition. Partitions can then be read from disk and joined using the original in-memory algorithm to produce the same output. This divides the original problem into N subproblems. 

Note that a GRACE hash join only works if the size of a single partition does not exceed the operator’s work memory, since the partitions must be read fully into memory. To work around this limitation, the algorithm can simply apply the same divide-and-conquer approach to the large partition if it gets too large (i.e., repartition). In edge cases, it is possible that tuples with the same join column exist, making it impossible for a partition to decrease in size, regardless of the number of repartition attempts. In this case, a partition is sorted and a *merge join* is used.

### Merge Join

A merge join outputs the same results as a hash join, but is only used when the inputs are already sorted by the equality columns. As was done with the `customers` in the hash join example, the merge join avoids the need to construct a hash table with one side of the input, making the operator more efficient. The merge join operator can simply advance both input streams until the tuples match on the equality columns, output the result of joining these tuples, and then move on to the next set of tuples that match on the equality columns.

The merge join operator is generally considered a streaming algorithm, since not much state needs to be buffered during a merge join. However, in the case where both input streams have many tuples with the same equality column value, the operator needs to buffer all of these tuples on at least one side, since the result will be a cross product of both sets of tuples. In this case, spilling to disk is very simple, as the only thing that is needed is an append-only log that will be replayed multiple times.

## The Building Block

All the algorithms covered so far have a common disk usage access pattern: they append data to on-disk queues (also known as an append-only log) and read from that queue sequentially (and possibly more than once).

At a high level, a caller can enqueue and dequeue columnar batches of tuples. It can also reset the queue to go back to dequeue from the front of the queue. Under the hood, batches are serialized, compressed, and appended to a file. If the file exceeds a certain size, the queue rolls over to a new file. An in-memory cursor is maintained and incremented as the caller reads from these files. This design, including alternatives, is covered more in depth in this [RFC.](https://github.com/cockroachdb/cockroach/blob/master/docs/RFCS/20191113_vectorized_external_storage.md)

Batches are written to disk using the [Apache Arrow IPC file format](https://arrow.apache.org/docs/format/Columnar.html#serialization-and-interprocess-communication-ipc), which is a specification for how to serialize columnar data. Although we don’t use Arrow batches to represent physical data in the vectorized execution engine directly, we use a very similar representation that can be easily and efficiently converted into Arrow batches and serialized as such.

For example, suppose we have a batch of strings that is represented using a flat bytes representation, composed of three buffers: 
- 
A null bitmap to represent any nulls.

- 
A single bytes buffer that represents all the strings.

- 
An accompanying offsets buffer that represents the start and end indices of the individual strings in the bytes buffer. 

These three buffers are cast to bytes, treated as Arrow buffers, and then serialized using the same [flatbuffer](https://github.com/google/flatbuffers) spec, which generally consists of some metadata that points to these buffers and the buffers themselves.

Using this physical representation avoids a copy by using an *O(1)* cast to bytes because the data is already contiguous in memory. If strings were represented as a two-dimensional array, the data would need to be prepared for serialization by allocating a new buffer and then iterating and copying each element into it.

## Conclusion

In this post, we covered how we used a single building block to implement a variety of on-disk algorithms in the vectorized execution engine in [CockroachDB v20.1.](https://www.cockroachlabs.com/get-cockroachdb/) Queries that could previously use an unbounded amount of memory now use up to a constant amount of work memory and spill to disk if this amount is not enough.

With the addition of disk spilling, we renamed the `experimental_on` vectorize mode to on, since we now consider the vectorized execution engine ready for production use although it is not yet fully enabled by default. As a reminder, only queries that use streaming (non-buffering) operators and that are likely to read more rows than the `vectorize_row_count_threshold` setting (which defaults to 1,000) are run by default in v20.1. By running `SET vectorize=on` in a session or `SET CLUSTER SETTING sql.defaults.vectorize=on`, all supported queries including ones that spill to disk will be run through the vectorized execution engine.

I hope you enjoyed learning about how we added disk spilling to the vectorized execution engine, and I urge you to try enabling it to speed up any complex joins or aggregations. 

And if you’re interested in working on similar projects, we’ve got good news: [Cockroach Labs is hiring](https://www.cockroachlabs.com/careers)!