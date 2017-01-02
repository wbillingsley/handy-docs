# RefCache

If a [`Latch`](latch) is a single-item asynchronous cache, a `RefCache` is a multi-item asynchronous cache.

Again, it contains listener methods so that on the client you can ensure that any invalidation of a cache entry will trigger a rerender.

# LookUpCache

A `LookUpCache` is a particular kind of cache for looking up IDs. In future versions, it will use `RefCache` but at the moment it's its own thing.

There are a few unique features we would like a `LookUpCache` to be able to do:

- If we give it a `LazyId[T, K]` we'd like it to see if it's already in the cache.
- If it's given some other kind of `Ref[T]` we'd like it to cache it against its ID.
- If it's given a `Seq[Id[T, K]]` or an `Ids[T, K]` we'd like it to be able to work out which ones are missing, bulk-fetch just those, and then return a `RefMany[T]` in the right order including the ones it had already cached.

That's not all in the current version, but will be in future ones.