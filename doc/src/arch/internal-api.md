# API

## Chunk API

ChunkStorage provides chunk access:

```rust
trait ChunkStorage {
    fn add(&mut self, chunk_id, bytes);
    fn remove(&mut self, chunk_id);
    fn read(&mut self, chunk_id) -> Stream<>
}
```

## CacheServer API

Server level API

```rust
trait Cache {
    /// Write chunks to an object.
    /// Nonexistent object will added implicitly
    fn write(&mut self, key, chunks)

    /// Read specified `range` of bytes in an object
    fn read(&mut self, key, range)
}
```
