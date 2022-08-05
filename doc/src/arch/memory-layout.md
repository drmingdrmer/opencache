# Memory layout

`CacheReplacement` policy and object index resides in memory as one single data structure `ObjectIndex`. It is used to lookup object by key and evict object.
(one of the famous  `CacheReplacement` is `LRU`).

In opencache, `ObjectIndex` is the only in-memory data.

When cache server restarts:
Object keys are loaded into memory from all ChunkIndex, 
And all Access log are replayed by the `ObjectIndex`.



## ObjectIndex


`ObjectIndex` has a `map` for locating an `entry` in it by entry key.
Cache entries are organized with one or more double linked list(`CacheReplacement`) to maintain access
history thus to decide which node to evict. 

The `map` in `ObjectIndex` is an index of cached object and non-cached objects(evicted but not cleand).
The value(cache entry) of `ObjectIndex` map is mainly the object metadata.

Object meta includes:
- Chunk file it resides on disk.
- Meta data such as offset, size

Entry key is the unique identifier of cached item.
In this project it is just the object key.

Every cached `Object` has a corresponding **entry** in
`CacheReplacement`.
**But not vice versa**.

- E.g., for `LRU`, it is a one-to-one mapping between lru-entry and an object:
  A `Object` that does not have a corresponding lru-entry must have been removed or be going to be removed.

- For `2Q` or `LIRS`, there may be an entry that has no corresponding `Object`.
    Such an entry represents an **accessed** but **NOT cached** `Object`, or an
    evicted but not yet cleaned `Object`.


```bob
     Bucket

     .--------------.             .-------.
     | Replacement  |             |       |
     |              |             |       |
     | Entry-1 <----|-------------| key-1 |
     | Entry-2 <----|-------------| key-2 |
     | ..           |             |   ..  |
     | Entry-i      |             |       |
     '--------------'             '-------'

```

## Capacity

The capacity of a cache system is the max **size** of all the content it can
cache.
E.g., for `LRU`, the capacity is the max number of entries it can have.

In our project, the capacity is the sum of all cached objects: ∑objectᵢ.size


## Read an object

The working flow for a `read` operation includes:

- Locate the Bucket by hash value of object key.

- Find the chunk file, size and offset in `ObjectIndex`,
  if it does not exist, or it exists but is marked as evcited, return 404.

    This is also an access to the `CacheReplacement` at the same time,
    thus it may trigger an eviction for an `Object`.
    If it does, remove it from `ObjectStore`.

- Load bytes from `ObjectStore` and respond it to client.

- Append this request to the access log.


```bob
  .--------.
  | client |
  '--------'
----->|
req   |
      |                          .---------------.
      | find chunk-id            | ObjectIndex   |   In memory
      |------------------------->|               |
      |                          | chunk_id_1=() |
      |                 chunk-id | chunk_id_2=() |
      |<-------------------------| ..            |
      |                          | chunk_id_i=ø  |
      |                          '---------------'
      |
      |
      | read chunk               .------------.
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


## Write an object

- Locate the `Bucket` for this object by the hash value of the key.

- Write the object to `ObjectStore`, by appending the object to the active chunk file
    and appending a key record to `ChunkIndex` file, no need to fsync

- Writing an object also implies a read on it.
  Thus the last step of writing an object is to update the `CacheReplacement`.


```bob
  .--------.
  | client |
  '--------'
----->|
req   |
      |                       .--------------------.
      |                       |   ObjectStore      |
      |                       |                    |
      | write object          |  .--------------.  |
      |-----------------------|->| Active Chunk |  |   On disk
      |                       |  |              |  |
      |                       |  | object1      |  |
      |                       |  | object2      |  |
      |                       |  | ..           |  |
      |                       |  '--------------'  |
      |                       |                    |
      |                       |  .--------------.  |
      |                       |  | Active       |  |   On disk
      |                       |  | ChunkIndex   |  |
      |                       |  |              |  |
      |                       |  | key1         |  |
      |                       |  | key2         |  |
      |                       |  '--------------'  |
      |                       '--------------------'
      |
      |                          .---------------.
      | Add object key           | ChunkIndex    |     In memory
      |------------------------->|               |
      |                          | chunk_id_1=() |
      |                          | chunk_id_2=() |
      |                          | ..            |
      |                          | chunk_id_i=ø  |
      |                          '---------------'
      |
      |
<-----|
  resp|
      |
      | append access log        .-------------.
      |------------------------->| AccessStore |       On disk
      |                          |             |
      |                          '-------------'
      v
      time

```
