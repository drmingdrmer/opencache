# Memory layout

KeyStore resides in memory for quick access: `ObjectIndex`.
Access data resides in memory too and is part of the `CacheReplacement`(one of the famous  `CacheReplacement` is `LRU`).
Chunks can be optionally cached in memory but it should be OK there is no in-memory chunk-cache.

There are two `CacheReplacement` in opencache:
`ObjectReplacement` tracks object level access history.
`ChunkReplacement` tracks chunk level access history.

When cache server restarts:
All of the keys are loaded into memory.
All Access data are replayed by the two `CacheReplacement` implementation.



## Cache replacement algorithm


`CacheReplacement`: cache replacement algorithm, e.g., `LRU`, `2Q`, `LIRS`.

`CacheReplacement` internally has a `map` for locating a node in it by a user
defined node key and one or more double linked list to maintain access history
thus to decide which node to evict.

- Node key is the identifier of cached item.
  In this project is either the `Key` of an object, or a `ChunkId`.

- Node is exactly the node in a linked list. A `Node` may have various properties
    in different replacement algorithms.

Every `Object` or `Chunk` has a corresponding **node** present in a replacement
algorithm.
**But not vice versa**.

- E.g., for `LRU` of `Chunk`, it is a one-to-one mapping between lru-node and `Chunk`.
  Every `Chunk` has a lru-node.
  A `Chunk` that does not have a corresponding lru-node must have been removed or be going to be removed.

- For `2Q` or `LIRS`, there may be a node that has no corresponding `Chunk`.
    Such a node represents an **accessed** but **NOT cached** `Chunk`, or an
    evicted but not yet cleaned `Chunk`.


```bob

          .-------------------.             .------------.
          | ObjectReplacement |             | Keys store |
          |                   |             |            |
          | Node-1 <----------|-------------|-- key-1    |
          | Node-2 <----------|-------------|-- key-2    |
          | ..                |             |   ..       |
          | Node-i            |             |            |
          |                   |             |            |
          '-------------------'             '------------'

```

## Capacity

The capacity of a cache system is the max **size** of all the content it caches.
E.g., for `LRU`, the capacity is the max number of `Node` it can have.

In our project, the capacity is the sum of all cached chunks: ∑chunk_sizeᵢ


## In memory data structures

`ObjectIndex` is:
- the in-memory index of cached keys and non-cached keys(evicted but not cleand).
- It is also the cache replacement algorithm implementation for keys, since a replacement
algorithm usually is a `hashmap` whose `value` are linked list node.


`ChunkIndex` is:
- the in-memory index of cached chunks and non-cached chunks(evicted but not cleand).
- It is also the cache replacement algorithm implementation for chunks.
  `ChunkIndex` is a `hashset` whose `key` are linked list node.


`ChunkLookup` is a lookup table to locate the file for a chunk by `ChunkId`.
It is a list of chunk file path and first chunk id in the chunk file, sorted by
first-chunk-id.
To find the chunk file path by `ChunkId`, run a binary search on `ChunkLookup`.
Thus `ChunkLookup` is very similar to a big root node in a btree.


## Read an object

The working flow for a `read` operation includes:
- Find the object by key in `ObjectIndex`, extract `ChunkId`s by the
    range of bytes to read.

- Filter out `ChunkId` that does not exist.

- Load bytes from `ChunkStore` and respond it to client.

- Record access log.


```bob
  .--------.
  | client |
  '--------'
----->|                          .-------------.
req   | lookup(key)              | ObjectIndex |     In memory
      |------------------------->|             |
      |                          | key_1=meta  |
      |      key meta(chunk-ids) | key_2=meta  |
      |<-------------------------| ..          |
      |                          | key_i=ø     |
      |                          '-------------'
      |
      |                          .---------------.
      | find non-null chunk-ids  | ChunkIndex    |   In memory
      |------------------------->|               |
      |                          | chunk_id_1=() |
      |                chunk-ids | chunk_id_2=() |
      |<-------------------------| ..            |
      |                          | chunk_id_i=ø  |
      |                          '---------------'
      |
      |                          .----------------.
      | get chunk file paths     | ChunkLookup    |  In memory
      |------------------------->|                |
      |                          | chunk_id_i=fn1 |
      | file paths               | ..             |
      |<-------------------------| chunk_id_j=fn2 |
      |                          | ..             |
      |                          '----------------'
      |
      | read chunks              .------------.
      |------------------------->| ChunkStore |      On disk
      |                          |            |
      |                          | chunk1     |
<-----|                          | chunk2     |
  resp|                          | ..         |
      |                          | chunki     |
      |                          '------------'
      |
      | append access log        .-------------.
      |------------------------->| AccessStore |      On disk
      |                          |             |
      |                          '-------------'
      v
      time

```
