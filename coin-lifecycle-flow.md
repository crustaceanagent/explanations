# Bitcoin Core Coin Lifecycle: From AddCoin to Disk

This document explains the lifecycle of coins in Bitcoin Core — from the moment `AddCoin` is called to when the coin is written to disk. It covers the nomenclature, relevant functions, and the circumstances under which coins are persisted.

---

## 1. Core Data Structures

### File: `src/coins.h`

#### `Coin` class (line 34)

Represents an unspent transaction output (UTXO):

```cpp
class Coin {
public:
    CTxOut out;           // The output: amount + scriptPubKey
    unsigned int fCoinBase : 1;  // Was this from a coinbase tx?
    uint32_t nHeight : 31;       // Block height where tx was included

    bool IsSpent() const { return out.IsNull(); }
};
```

**Nomenclature:**
- A `Coin` is **spent** when its `out` is null (cleared)
- **Coinbase** coins come from block reward transactions
- **nHeight** tracks when the coin was created (block inclusion height)

---

### File: `src/coins.h`

#### `CCoinsCacheEntry` struct (line 111)

A coin in the cache with metadata flags:

```cpp
struct CCoinsCacheEntry {
    Coin coin;  // The actual coin data

    enum Flags {
        DIRTY = (1 << 0),  // Potentially different from parent cache
        FRESH = (1 << 1),  // Parent doesn't have this coin
    };
};
```

**Valid states:**
| State | Description |
|-------|-------------|
| unspent, FRESH, DIRTY | New coin created in cache |
| unspent, not FRESH, DIRTY | Coin changed during reorg |
| unspent, not FRESH, not DIRTY | Unspent coin from parent cache |
| spent, not FRESH, DIRTY | Spent coin needs to flush spentness |

---

### File: `src/coins.cpp`

#### `CCoinsViewCache` class

A caching layer over a base `CCoinsView`. Coins are added to an in-memory cache first:

```cpp
class CCoinsViewCache {
    CCoinsMap cacheCoins;  // In-memory cache: COutPoint -> CCoinsCacheEntry
    CCoinsView* base;      // Parent view (usually the database)
};
```

---

## 2. AddCoin: The Entry Point

### File: `src/coins.cpp`

#### `CCoinsViewCache::AddCoin()` (line 89)

```cpp
void CCoinsViewCache::AddCoin(const COutPoint &outpoint, Coin&& coin, bool possible_overwrite) {
    assert(!coin.IsSpent());
    if (coin.out.scriptPubKey.IsUnspendable()) return;

    // Get or create entry in cache
    auto [it, inserted] = cacheCoins.emplace(outpoint, CCoinsCacheEntry{});
    
    // Determine if FRESH (new coin not in parent)
    bool fresh = false;
    if (!possible_overwrite) {
        if (!it->second.coin.IsSpent()) {
            throw std::logic_error("Attempted to overwrite an unspent coin");
        }
        fresh = !it->second.IsDirty();  // Can be fresh if parent doesn't have it
    }

    // Move coin into cache
    it->second.coin = std::move(coin);
    CCoinsCacheEntry::SetDirty(*it, m_sentinel);   // Mark as DIRTY
    ++m_dirty_count;

    if (fresh) CCoinsCacheEntry::SetFresh(*it, m_sentinel);  // Mark as FRESH
}
```

**What happens:**
1. **Allocates** a cache entry for the outpoint (or gets existing)
2. **Marks** the entry as DIRTY (needs to be flushed to parent)
3. **Optionally marks** as FRESH if it's a new coin not in parent
4. **Increments** dirty count

**The coin is NOT written to disk here** — it's only added to the in-memory cache.

---

## 3. Higher-Level: AddCoins

### File: `src/coins.cpp`

#### `AddCoins()` (line 144)

Wrapper that adds all outputs of a transaction:

```cpp
void AddCoins(CCoinsViewCache& cache, const CTransaction &tx, int nHeight, bool check_for_overwrite) {
    bool fCoinbase = tx.IsCoinBase();
    const Txid& txid = tx.GetHash();
    
    for (size_t i = 0; i < tx.vout.size(); ++i) {
        bool overwrite = check_for_overwrite ? cache.HaveCoin(COutPoint(txid, i)) : fCoinbase;
        cache.AddCoin(COutPoint(txid, i), Coin(tx.vout[i], nHeight, fCoinbase), overwrite);
    }
}
```

---

## 4. Where AddCoin is Called

### File: `src/validation.cpp`

#### During block connection — `UpdateCoins()` (line 1996)

Called when a block is connected to the chain:

```cpp
void UpdateCoins(const CTransaction& tx, CCoinsViewCache& inputs, CTxUndo &txundo, int nHeight) {
    // Mark inputs spent
    if (!tx.IsCoinBase()) {
        for (const CTxIn &txin : tx.vin) {
            inputs.SpendCoin(txin.prevout, &txundo.vprevout.back());
        }
    }
    // Add outputs as new coins
    AddCoins(inputs, tx, nHeight);
}
```

#### During replay — `ReplayBlocks()` (line 4764)

Used when recovering from an interrupted flush:

```cpp
AddCoins(inputs, *tx, pindex->nHeight, true);  // check = true for overwrite
```

---

## 5. When Are Coins Written to Disk?

Coins are **not** written immediately when `AddCoin` is called. They're flushed to the LevelDB database under specific circumstances.

### File: `src/validation.cpp`

#### `FlushStateToDisk()` (line 2699)

The main flush function. A write to disk occurs when:

| Condition | Trigger |
|-----------|---------|
| `FORCE_SYNC` | Explicit sync request |
| `FORCE_FLUSH` | Cache critical, must flush |
| `PERIODIC` + cache ≥ LARGE | Cache > 90% of limit |
| `IF_NEEDED` + cache ≥ CRITICAL | Cache exceeded limit |
| `PERIODIC` + time elapsed | 50-70 minutes since last write |
| Prune mode | Block files being pruned |

#### Cache size thresholds (`src/validation.h`):

```cpp
constexpr int64_t LargeCoinsCacheThreshold(int64_t total_space) noexcept {
    // LARGE: cache > 90% of available space
    // CRITICAL: cache exceeds total space
    return std::max((total_space * 9) / 10,
                    total_space - MAX_BLOCK_COINSDB_USAGE_BYTES);
}
```

---

## 6. The Flush Process

### Step 1: CoinsViewCache::Flush() or Sync()

### File: `src/coins.cpp`

```cpp
void CCoinsViewCache::Flush(bool reallocate_cache) {
    auto cursor{...};
    base->BatchWrite(cursor, hashBlock);  // Write to parent
    cacheCoins.clear();                    // Clear cache
}
```

- **Flush()**: Writes and clears the cache
- **Sync()**: Writes and clears flags (keeps entries in memory)

### Step 2: CCoinsViewDB::BatchWrite()

### File: `src/txdb.cpp`

```cpp
void CCoinsViewDB::BatchWrite(CoinsViewCacheCursor& cursor, const uint256& hashBlock) {
    CDBBatch batch(*m_db);
    
    // Write head blocks (for crash recovery)
    batch.Erase(DB_HEAD_BLOCKS);
    batch.Write(DB_HEAD_BLOCKS, Vector(hashBlock, old_tip));
    
    for (auto it{cursor.Begin()}; it != cursor.End(); it = cursor.NextAndMaybeErase(*it)) {
        if (it->second.IsDirty()) {
            CoinEntry entry(&it->first);
            if (it->second.coin.IsSpent()) {
                batch.Erase(entry);      // Remove spent coins
            } else {
                batch.Write(entry, it->second.coin);  // Write unspent
            }
        }
    }
    
    // Mark database consistent
    batch.Erase(DB_HEAD_BLOCKS);
    batch.Write(DB_BEST_BLOCK, hashBlock);
    
    m_db->WriteBatch(batch);  // Atomic write to LevelDB
}
```

### Step 3: CoinEntry Serialization

### File: `src/txdb.cpp`

```cpp
struct CoinEntry {
    COutPoint* outpoint;    // txid + output index
    uint8_t key{DB_COIN};   // Namespace key
    
    SERIALIZE_METHODS(CoinEntry, obj) { 
        READWRITE(obj.key, obj.outpoint->hash, VARINT(obj.outpoint->n)); 
    }
};
```

**Key format:** `COIN` + txid + output_index

**Value:** Serialized `Coin` object (height + coinbase flag + CTxOut)

---

## 7. Summary: Coin Lifecycle

```
AddCoin() called
       │
       ▼
┌─────────────────────────────────────┐
│  Add to in-memory cache (CCoinsMap) │
│  - Mark DIRTY                       │
│  - Mark FRESH (if new)              │
│  - Increment dirty count            │
└─────────────────────────────────────┘
       │
       ▼
  Block connected / other operations
       │
       ▼
  FlushStateToDisk() triggered?
       │
  ┌────┴────┐
  │         │
  NO        YES
  │         │
  ▼         ▼
In memory  BatchWrite to LevelDB
           │
           ├─► Erase(DB_HEAD_BLOCKS)  // Start atomic batch
           ├─► Write(DB_HEAD_BLOCKS, [new_tip, old_tip])
           │
           ├─► For each DIRTY entry:
           │     ├─► IsSpent? → Erase
           │     └─► Unspent? → Write(CoinEntry, Coin)
           │
           ├─► Erase(DB_HEAD_BLOCKS)
           ├─► Write(DB_BEST_BLOCK, new_tip)
           │
           └─► m_db->WriteBatch()  // Atomic commit
```

---

## 8. Key Functions Summary

| Function | File | Purpose |
|----------|------|---------|
| `AddCoin()` | `coins.cpp` | Add coin to in-memory cache |
| `AddCoins()` | `coins.cpp` | Add all tx outputs to cache |
| `UpdateCoins()` | `validation.cpp` | Add coins during block connection |
| `SpendCoin()` | `coins.cpp` | Mark coin as spent in cache |
| `Flush()` | `coins.cpp` | Write cache to parent view |
| `Sync()` | `coins.cpp` | Write and clear DIRTY flags |
| `FlushStateToDisk()` | `validation.cpp` | Orchestrate full flush |
| `BatchWrite()` | `txdb.cpp` | Write to LevelDB |

---

## 9. Nomenclature Summary

| Term | Meaning |
|------|---------|
| **Coin** | An unspent transaction output (CTxOut + metadata) |
| **CCoinsViewCache** | In-memory cache layer over coins database |
| **DIRTY** | Flag: coin modified relative to parent cache |
| **FRESH** | Flag: coin doesn't exist in parent cache |
| **Spent** | Coin's output is null (consumed) |
| **Coinbase** | Flag: coin from block generation transaction |
| **nHeight** | Block height where the containing transaction was included |
| **COutPoint** | Reference to a specific output (txid + index) |
| **CTxOut** | Transaction output (amount + scriptPubKey) |

---

## 10. Disk Persistence Triggers

Coins are written to disk (LevelDB) when:

1. **Cache full**: Coins usage exceeds configured limit (default ~450MB)
2. **Periodic flush**: 50-70 minutes have passed since last write
3. **Block processed**: After connecting a block (IF_NEEDED mode)
4. **Pruning**: When block files are pruned
5. **Shutdown**: During graceful shutdown (FORCE_SYNC)

The FRESH flag optimization: if a coin is FRESH and then spent before flush, it can be deleted entirely without ever writing to disk.
