---
title: How it broke - Cache stampede
date: 2024-10-07
permalink: /cache-stampede
---

Cache invalidation is one of the [Two Hard Things](https://martinfowler.com/bliki/TwoHardThings.html) of software engineering (along with naming things). When working with a cache, it seems inevitable you'll end up at least thinking about it. This

## Situation

Every second, a couple dozen concurrent requests were made to an API server. To properly respond to these requests, the server needed to check whether items it serves are anomalous. Whether items are anomalous is kept track of by some other service (`Item API`), which has a HTTP GET-endpoint to retrieve the set of anomalous items.

If this cache had been large, it probably would have ended up as a Redis cache or some specialized in-memory database such as RocksDB, but this particular set only contained a couple of items so a Python object was all we needed.

The initial implementation looked a bit like this simplified example:

```python
from datetime import timedelta, datetime

@dataclass
class AnomalousItemCache:
    item_ids: set[str] | None = None

class ItemApiClient:
    """Client for interacting with the Item API"""
    def __init__(self, cache_validity_seconds: int, cache: AnomalousItemCache):
        ...
        self._cache_lifetime = timedelta(seconds=cache_validity_seconds)
        self._last_cache_update = datetime.now() - timedelta(seconds=cache_validity_seconds)
        self._anomalous_item_cache = cache

    async def item_is_anomalous(self, item_id: int) -> bool:
        if self._anomalous_item_cache is None or self._cache_expired():
            # Perform the HTTP request to get the items and update the cache
            ...

        return item_id in self._anomalous_item_cache

    def _cache_expired(self) -> bool:
        return datetime.now() - self._last_cache_update >= self._cache_lifetime
```

## Problem

This worked just fine, except for when the cache expired (or, in other words: the cache was _invalidated_). Starting from that moment, any request to the server that made use of this code would start performing HTTP requests to the `Item API`. This behavior is called a `cache stampede`: the gate opens and the requests storm out.

The server was handling many more parallel requests than the `Item API` was built to handle; therefore this overloaded the `Item API`, causing it to return 503-error responses.

(The HTTP request to the `Item API` had some error handling (not shown in the example above), so that the cache would be used even it had expired in case of an error response from `Item API`. Because of this, the issue had little impact on users. It did result in error logs and failed requests that cluttered our monitoring systems, so it needed to be resolved.)

## Solution

The solution we ended up implementing was a locking mechanism for the cache. In a nutshell:

- Only one process tries to update the cache at a time by acquiring a lock;
- Other processes that try to access the cache afterwards see that it is locked and use the expired cache in the meantime.

There's one more (somewhat theoretical) situation that needs to be handled: the cache could be locked by process A in the time between process B checking if the cache is locked and process B attempting to acquire the lock. If this is all we do, the cache will be updated by process B as well. To prevent this unnecessary update, process B should check again if the cache has expired once it acquires the lock.

The solution we implemented is similar to this simplified example:

```python
import asyncio
from datetime import timedelta, datetime

@dataclass
class AnomalousItemCache:
    item_ids: set[str] | None = None

class ItemApiClient:
    """Client for interacting with the Item API"""
    def __init__(self, cache_validity_seconds: int, cache: AnomalousItemCache):
        self._cache_lifetime = timedelta(seconds=cache_validity_seconds)
        self._last_cache_update = datetime.now() - timedelta(seconds=cache_validity_seconds)
        self._anomalous_item_cache = cache
        self._cache_lock = asyncio.Lock()

    async def item_is_anomalous(self, item_id: int) -> bool:
        if not (self._anomalous_item_cache is None or self._cache_expired()) or self._cache_lock.locked():
            return item_id in self._anomalous_item_cache

        async with self._cache_lock:  # Awaits acquisition of the lock
            # In case the cache was filled while waiting for the lock, check cache validity again
            if self._anomalous_item_cache is None or self._cache_expired():
                return item_id in self._anomalous_item_cache

            # Perform the HTTP request to get the items and update the cache
            ...

        return item_id in self._anomalous_item_cache

    def _cache_expired(self) -> bool:
        return datetime.now() - self._last_cache_update >= self._cache_lifetime
```

## Resources & further reading

- [A Complete Beginner Guide for Cache Penetration, Stampede, Avalanche](https://github.com/FormoSeanIap/redis-cache-penetration-stampede-avalanche)
