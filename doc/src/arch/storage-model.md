# Storage model

CacheServer stores data in chunks(a byte stream identified by a unique utf-8 string key).

## Chunk

A `Chunk` is barely a byte slice.

## Chunk group

Chunks with different sizes are stored in different chunk groups(`ChunkGroup`), e.g.:
- `(0, 4KB]`
- `(4KB, 1MB]`
- `(1MB, 8MB]`
- `(8MB, +oo]`

Opencache has several predefined ChunkGroup data is stored in the smallest
predefined chunk size that is no less than the specified one.


# Data types

## Chunk group id

`ChunkGroupId` is 2 ascii char in file name, or 1 byte in memory.
`ChunkGroupId` is a encoded representation of the size of chunks in it, similar
to float number representation.

The first char(or 4 higher bits) is the `power`.
The second char(or 4 lower bits) is the `significand`.

```
        power          significand
ascii   0              1
bits    4 higher bits  4 lower bits
```

The chunk size is calculated as follow: `significand * 16^power`

E.g. chunk group id of `48K` chunks is `3c`: `12 * 16^3 = 48K  // c=12`;

Max chunk group id is `ff`: `15 * 16^15 = 15E`

Several ChunkGroupId examples are listed as below:

```
01: 2 * 16^0 = 2        08: 8 * 16^0 = 8
11: 2 * 16^1 = 32       18: 8 * 16^1 = 128
21: 2 * 16^2 = 512      28: 8 * 16^2 = 2K
31: 2 * 16^3 = 8K       38: 8 * 16^3 = 32K
41: 2 * 16^4 = 128K     48: 8 * 16^4 = 512K
51: 2 * 16^5 = 2M       58: 8 * 16^5 = 8M
61: 2 * 16^6 = 32M      68: 8 * 16^6 = 128M
71: 2 * 16^7 = 512M     78: 8 * 16^7 = 2G
```

## Chunk id

`ChunkId` is a `u64`,
in which the most significant byte is `ChunkGroupId`,
the other 7 bytes is mono incremental index in 54-bit int.

```
       <ChunkGroupId> <index>
byte:  [0]            [1, 7]
```


## Access entry

A cache replacement algorithm depends the access log to decide which
chunk should be evicted.
Access log is barely a series of `ChunkId`.


# Storage layout

Opencache on-disk storage include:
- Manifest file that track all other kind of on-disk files.
- Chunk files to store cached file data.


## Manifest file

Manifest is a snapshot of current storage.
I.e., it's a list of files belonging to this snapshot.

```yaml
data_ver: 0.1.0
access:
  acc-1, acc-2...
buckets:
    bucket_1:
        chunk-1,  # the first is active
        chunk-2, 
    bucket_2:
        chunk-3,  # the first is active
        chunk-4, 
    ...
```

## Object store

ObjectStore puts object into several `Bucket`s by hash value of object key.

Every Bucket consists of **one** active(writable) chunk file and several
closed(readonly) chunk files. The active chunk file is append only.

Write is done by appending an `Object` into active chunk and appending a `Key`
into the index file of the active chunk.

Deleting is done by appending an tombstone Key.

- For an active chunk, ChunkIndex only contains removed key;
- For an closed chunk, ChunkIndex contains present keys and removed keys.

When restarting, keys are loaded from ChunkIndex into memory.


## Access store

Opencache evicts object by accessing history.
Thus the sequence of recent access has to be persisted.
Otherwise when a cache server restarts, cached data will be evicted
in an unexpected way, due to lacking of accessing information.

Accessing data is stored in several append only files.

Old access data will be removed periodically.
Access data does not need compact.


```bob
      .----------.
      | Manifest |
      '----------'


  Access      ObjectStore
              |
              +-------------------------------+---..
              |                               |
              Bucket-1                        Bucket-2   ..

  .----.      Active            Key
  | k1 |      .----.  Chunk     .------.
  | k2 |      |obj1|  Index  .->|key   |
  | .. |    .-|obj2|  .----. |  |add/rm|
  |CKSM|    | | .. |<-| k5-|-'  |offset|
  | ki |    | | .. |  | k6 |    |size  |
  | .. |    | '----'  |CHSM|    |ts    |
  '----'    |   ..    | .. |    '------'
    ..      |         '----'    
  .----.    '------------------>Object   
  | k1 |                        .-------.
  | k2 |      Compacted         | ver   |
  | .. |                        | key   |
  |CKSM|      .----.  Chunk     | bytes |
  | ki |      |obj1|  Index     | CKSM  |           
  | .. |      |obj2|  .----.    '-------'           
  '----'      | .. |<-| k5 |
              | .. |  | k6 |
              '----'  | .. |
                      |CHSM|
                      '----'
               ..      

```


# Operations

## Write

A cache server does not need strict data durability, thus when new data is
written(object or access data), it does not have to fsync at once.

For different kind of data, the fsync policies are:

- Access and ChunkIndex: are fsync-ed for every `k MB`(where `k` is a configurable parameter). A checksum
  record has to be written before fsync. Since on disk data still has to be
  internally consistent.  The checksum record contains checksum of the `k MB`
  data except itself.

  In other words, Access and ChunkIndex files are split into several `k MB` segment.

- Chunks is different: Every chunk contains a checksum of itself.
    Chunk files is fsync-ed periodically.


## Update manifest

A write operation does not update `Manifest` file.
`Manifest` is only replaced when files(Chunk files or access store files)
are added or removed(or configuration changed).
E.g., when a new chunk file is opened, or a compaction of Chunk files is done.

The Manifest file is named with monotonic incremental integers, e.g.,
`manifest-000021`.

Manifest files can only be added or removed.
When restarting, opencache list all of the manifest file and use the last valid one(by checking the
checksum).
