# Collection Recreation Bug - Complete Logical Analysis

## Bug Symptom
After creating a collection, adding items, deleting items and collection, attempting to recreate the collection fails with a "not found" error. The error occurs during index operations. After service restart, recreation works.

## Root Cause: Stale IndexCacheManager Cache

### The Architecture

The codebase has two index insertion strategies:
1. **SimpleIndexInserter**: Creates one index per collection upfront
2. **DatetimeIndexInserter**: Creates datetime-partitioned indices dynamically

Both strategies interact with a caching layer called `IndexCacheManager`.

### The Caching Mechanism

**File**: `stac_fastapi/sfeos_helpers/search_engine/selection/cache_manager.py`

```python
class IndexCacheManager:
    def __init__(self, cache_ttl_seconds: int = 3600):  # 1-hour TTL
        self._cache: Optional[Dict[str, List[str]]] = None
        self._timestamp: float = 0
        self._ttl = cache_ttl_seconds
```

**File**: `stac_fastapi/sfeos_helpers/search_engine/selection/selectors.py`

```python
class DatetimeBasedIndexSelector(BaseIndexSelector):
    _instance = None  # SINGLETON - persists across requests

    def __init__(self, client: Any):
        if not hasattr(self, "_initialized"):
            self.cache_manager = IndexCacheManager()  # 1-hour cache
            self.alias_loader = IndexAliasLoader(client, self.cache_manager)
```

Key properties:
- **Singleton pattern**: One instance per process lifetime
- **1-hour TTL**: Cache persists for up to 3600 seconds
- **No invalidation on collection deletion**: Cache is never explicitly cleared

### The Cache Content

**File**: `cache_manager.py:74-96`

```python
async def load_aliases(self) -> Dict[str, List[str]]:
    response = await self.client.indices.get_alias(index=f"{ITEMS_INDEX_PREFIX}*")
    result = defaultdict(list)
    for index_info in response.values():
        aliases = index_info.get("aliases", {})
        items_aliases = sorted([
            alias for alias in aliases.keys()
            if alias.startswith(ITEMS_INDEX_PREFIX)
        ])
        if items_aliases:
            result[items_aliases[0]].extend(items_aliases[1:])
    return result
```

The cache stores a mapping like:
```python
{
    "items_test": ["items_test_2024-01-01", "items_test_2024-02-01"],
    "items_other": ["items_other_2024-01-01"]
}
```

This represents the **actual state of Elasticsearch/OpenSearch indices** at the time of caching.

### The Critical Code Path

**File**: `inserters.py:138-145` (DatetimeIndexInserter)

```python
async def _get_target_index_internal(self, index_selector, collection_id, product, check_size=True):
    # Line 138: Query what indices exist for this collection
    all_indexes = await index_selector.get_collection_indexes(collection_id)

    # Line 140-143: If no indices exist, handle as NEW collection
    if not all_indexes:
        target_index = await self.datetime_manager.handle_new_collection(
            collection_id, product_datetime
        )
        await index_selector.refresh_cache()
        return target_index

    # Line 147+: If indices exist, use them
    all_indexes.sort()
    start_date = extract_date(product_datetime)
    end_date = extract_first_date_from_index(all_indexes[0])
    # ... attempts to use all_indexes[0] which may not exist!
```

**File**: `cache_manager.py:117-127`

```python
async def get_collection_indexes(self, collection_id: str) -> List[str]:
    aliases = await self.get_aliases()  # Gets CACHED data if available
    return aliases.get(index_alias_by_collection_id(collection_id), [])
```

**File**: `cache_manager.py:98-107`

```python
async def get_aliases(self) -> Dict[str, List[str]]:
    cached = self.cache_manager.get_cache()
    if cached is not None:  # If cache exists and not expired
        return cached  # Return stale data!
    return await self.load_aliases()
```

### The Bug Sequence (Step-by-Step)

#### Step 1: Initial Collection Creation + Item Addition
```
Action: Create collection "test", add item with datetime "2024-01-01"

Flow:
1. Collection document created in COLLECTIONS_INDEX
2. Item insertion triggers _get_target_index_internal()
3. all_indexes = get_collection_indexes("test") → returns []
4. Code path: "if not all_indexes" → creates NEW index "items_test_2024-01-01"
5. IndexCacheManager.load_aliases() called
6. Cache populated: {"items_test": ["items_test_2024-01-01"]}
7. Cache timestamp set, TTL = 1 hour

Result: ✅ Item successfully added
Elasticsearch state: Index "items_test_2024-01-01" exists
Cache state: {"items_test": ["items_test_2024-01-01"]}
```

#### Step 2: Item Deletion
```
Action: Delete item from collection

Flow:
1. Item document removed from index "items_test_2024-01-01"
2. Index still exists (empty)

Result: ✅ Item deleted
Elasticsearch state: Index "items_test_2024-01-01" exists (empty)
Cache state: {"items_test": ["items_test_2024-01-01"]} (unchanged)
```

#### Step 3: Collection Deletion
```
Action: Delete collection "test"

Flow (database_logic.py:1568-1577):
1. Collection document deleted from COLLECTIONS_INDEX
2. delete_item_index(collection_id) called
3. Resolves alias "items_test" → finds index "items_test_2024-01-01"
4. Deletes alias and index from Elasticsearch
5. IndexCacheManager cache is NOT cleared (BUG!)

Result: ✅ Collection deleted from database
Elasticsearch state: NO indices for "test" collection
Cache state: {"items_test": ["items_test_2024-01-01"]} ⚠️ STALE!
```

#### Step 4: Collection Recreation
```
Action: Recreate collection "test"

Flow:
1. Collection document created in COLLECTIONS_INDEX
2. For DatetimeIndexInserter, no item index created yet

Result: ✅ Collection metadata created
Elasticsearch state: Collection exists, but NO item indices yet
Cache state: {"items_test": ["items_test_2024-01-01"]} ⚠️ STILL STALE!
```

#### Step 5: Attempt to Add Item to Recreated Collection (BUG TRIGGERS)
```
Action: Add item with datetime "2024-01-01" to recreated collection

Flow (inserters.py:138-145):
1. _get_target_index_internal() called
2. all_indexes = get_collection_indexes("test")
3. get_aliases() called
4. Cache check: cached data exists and TTL not expired (< 1 hour since Step 1)
5. Returns STALE cached data: {"items_test": ["items_test_2024-01-01"]}
6. all_indexes = ["items_test_2024-01-01"]
7. Code evaluates: "if not all_indexes" → FALSE (list is not empty!)
8. Code path: Treats collection as EXISTING, not NEW
9. Line 147: all_indexes.sort()
10. Line 149: end_date = extract_first_date_from_index(all_indexes[0])
11. Attempts to query or access "items_test_2024-01-01"
12. Elasticsearch returns: 404 NOT FOUND (index was deleted in Step 3!)

Result: ❌ "Not Found" Error
Elasticsearch state: Index "items_test_2024-01-01" DOES NOT EXIST
Cache state: {"items_test": ["items_test_2024-01-01"]} (cache lies!)
```

### Why Service Restart Fixes It

When the Python service restarts:
1. All Python objects are destroyed
2. The `DatetimeBasedIndexSelector` singleton `_instance = None`
3. The `IndexCacheManager` cache is cleared (object destroyed)
4. Next request creates fresh instance with empty cache
5. `get_aliases()` calls `load_aliases()` which queries Elasticsearch
6. Cache populated with CURRENT state (no indices for "test")
7. `all_indexes` correctly returns `[]`
8. Code correctly handles as NEW collection

### The Fix Logic

**File**: `database_logic.py` (both OpenSearch and Elasticsearch)

```python
async def delete_collection(self, collection_id: str, **kwargs: Any):
    # ... existing deletion logic ...
    await delete_item_index(collection_id)

    # Clear the @lru_cache to prevent stale values
    index_alias_by_collection_id.cache_clear()
    index_by_collection_id.cache_clear()

    # CRITICAL: Clear the IndexCacheManager singleton cache
    if DatetimeBasedIndexSelector._instance is not None:
        DatetimeBasedIndexSelector._instance.cache_manager.clear_cache()
```

**Why this works**:
1. After deleting indices from Elasticsearch, we clear the IndexCacheManager
2. Next call to `get_aliases()` finds empty cache
3. Calls `load_aliases()` which queries CURRENT Elasticsearch state
4. Returns empty mapping (no indices exist)
5. `get_collection_indexes("test")` returns `[]`
6. Code correctly evaluates `if not all_indexes` → TRUE
7. Handles as NEW collection
8. Creates fresh indices

### Why LRU Cache Clearing is Included

While `index_by_collection_id()` and `index_alias_by_collection_id()` are deterministic functions, they are used as **lookup keys** into the IndexCacheManager cache:

```python
async def get_collection_indexes(self, collection_id: str) -> List[str]:
    aliases = await self.get_aliases()
    # Uses the cached function as a dict key lookup
    return aliases.get(index_alias_by_collection_id(collection_id), [])
```

Clearing these caches doesn't change the generated names, but it ensures:
1. **Defensive programming**: Eliminates any potential cache-related edge cases
2. **Consistency**: All caches cleared together after deletion
3. **Minimal cost**: Cache rebuilds on next access (microseconds)

## Conclusion

The bug is caused by the **IndexCacheManager singleton** caching Elasticsearch index/alias mappings with a 1-hour TTL. When collections are deleted, the cache is not invalidated, causing it to return references to deleted indices. This causes "not found" errors when attempting to use those indices during recreation.

The fix is logically sound: **clearing the IndexCacheManager cache on collection deletion ensures the cache reflects current Elasticsearch state**.
