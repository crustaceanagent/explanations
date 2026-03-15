# Bitcoin Core AssumeUTXO: Snapshot Loading and Chainstate Initialization

AssumeUTXO is a feature that allows bitcoind to skip replaying the entire blockchain by loading a serialized UTXO set snapshot. This document details the snapshot format, loading process, and chainstate initialization.

---

## 1. Snapshot Format

### File Structure

A UTXO snapshot file (typically `utxo.dat`) has the following structure:

```
┌─────────────────────────────────────────┐
│  Magic Bytes (5 bytes)                  │  "utxo\xff"
│  Version (2 bytes, little-endian)       │  Currently 2
│  Network Magic (4 bytes)                │  e.g. 0xf9beb4d9 (mainnet)
│  Base Block Hash (32 bytes)             │  Hash of snapshot base block
│  Coins Count (8 bytes, varint)          │  Number of coins in UTXO set
├─────────────────────────────────────────┤
│  Coin Data (repeated):                  │
│    - Txid (32 bytes)                    │  Transaction ID
│    - Output Index (compact size)        │  Output position (0-indexed)
│    - Coin (serialized)                  │  {nHeight, out amount, scriptPubKey}
└─────────────────────────────────────────┘
```

### File: `src/node/utxo_snapshot.hpp`

#### `SNAPSHOT_MAGIC_BYTES`
```cpp
static constexpr std::array<uint8_t, 5> SNAPSHOT_MAGIC_BYTES = {'u', 't', 'x', 'o', 0xff};
```

**Validation:**
- Magic bytes: `"utxo\xff"` 
- Version: Currently `2` (supported versions stored in `m_supported_versions`)
- Network magic: Must match node's network (mainnet, testnet, signet, etc.)

---

### Snapshot Metadata (`SnapshotMetadata` class)

**Fields:**
| Field | Type | Description |
|-------|------|-------------|
| `m_base_blockhash` | uint256 | Hash of block that snapshot represents |
| `m_coins_count` | uint64_t | Number of coins in snapshot |

**Serialization format (src/node/utxo_snapshot.hpp):**
```cpp
template <typename Stream>
inline void Serialize(Stream& s) const {
    s << SNAPSHOT_MAGIC_BYTES;
    s << VERSION;
    s << m_network_magic;
    s << m_base_blockhash;
    s << m_coins_count;
}
```

---

### Coin Serialization Format

After metadata, each UTXO is serialized as:

1. **Txid** (32 bytes) - Transaction ID
2. **Output index** (compact size) - `n` in `OutPoint(hash, n)`
3. **Coin** - Serialized `Coin` structure containing:
   - `nHeight` - Block height at which output was created
   - `out` - `CTxOut` containing:
     - `nValue` - Satoshi amount
     - `scriptPubKey` - Output script

---

## 2. Loading a Snapshot at Runtime

### File: `src/rpc/blockchain.cpp`

#### `loadtxoutset` RPC (line 3346)
User-initiated snapshot loading via RPC.

```cpp
static RPCHelpMan loadtxoutset()
```

**What it does:**
1. Opens snapshot file at given path
2. Reads `SnapshotMetadata` from file header
3. Calls `ChainstateManager::ActivateSnapshot()` to load snapshot
4. Updates local services (removes `NODE_NETWORK`, adds `NODE_NETWORK_LIMITED`)

**Example:**
```bash
bitcoin-cli loadtxoutset utxo.dat
```

---

## 3. Snapshot Activation

### File: `src/validation.cpp`

#### `ChainstateManager::ActivateSnapshot()` (line 5571)
Main entry point for activating a UTXO snapshot.

```cpp
util::Result<CBlockIndex*> ChainstateManager::ActivateSnapshot(
    AutoFile& coins_file,
    const SnapshotMetadata& metadata,
    bool in_memory)
```

**What it does:**
1. **Validation:**
   - Checks no snapshot already loaded
   - Verifies base block hash is in `AssumeutxoForBlockhash` (known snapshot)
   - Looks up block index for base block via `LookupBlockIndex()`
   - Ensures block is not marked invalid
   - Verifies headers chain contains the base block
   - Checks mempool is empty

2. **Cache allocation:**
   - Resizes caches: 99% to snapshot chainstate, 1% to IBD chainstate

3. **Creates snapshot chainstate:**
   - Creates new `Chainstate` with `m_from_snapshot_blockhash = base_blockhash`
   - Calls `AddChainstate()` to add to chainstates

4. **Loads coins:**
   - Calls `PopulateAndValidateSnapshot()` to deserialize UTXO set

5. **Post-load validation:**
   - Calls `FlushSnapshotToDisk()` to persist to LevelDB
   - Updates block index flags for snapshot blocks
   - Triggers cache rebalancing

---

## 4. Snapshot Population and Validation

### File: `src/validation.cpp`

#### `ChainstateManager::PopulateAndValidateSnapshot()` (line 5737)
Deserializes and validates the UTXO snapshot.

```cpp
util::Result<void> ChainstateManager::PopulateAndValidateSnapshot(
    Chainstate& snapshot_chainstate,
    AutoFile& coins_file,
    const SnapshotMetadata& metadata)
```

**What it does:**

1. **Looks up base block:**
   - Calls `LookupBlockIndex(base_blockhash)` to find snapshot start block
   - Gets base height from `pindex->nHeight`

2. **Validates base height:**
   - Calls `GetParams().AssumeutxoForHeight(base_height)` to get expected hash
   - Ensures snapshot height is a known/supported assumeutxo height

3. **Work comparison:**
   - Checks `snapshot_start_block->nChainWork >= ActiveTip()->nChainWork`

4. **Coin deserialization loop:**
   - For each coin:
     ```cpp
     Txid txid;
     coins_file >> txid;
     size_t coins_per_txid = ReadCompactSize(coins_file);
     
     for (size_t i = 0; i < coins_per_txid; i++) {
         COutPoint outpoint;
         Coin coin;
         outpoint.n = static_cast<uint32_t>(ReadCompactSize(coins_file));
         outpoint.hash = txid;
         coins_file >> coin;
         
         // Validation:
         // - coin.nHeight <= base_height
         // - outpoint.n within valid range
         // - MoneyRange(coin.out.nValue)
         
         coins_cache.EmplaceCoinInternalDANGER(std::move(outpoint), std::move(coin));
     }
     ```

5. **Periodic flushing:**
   - Every 120,000 coins, checks cache size
   - If cache critical: calls `FlushSnapshotToDisk()`

6. **Finalization:**
   - Sets best block to `base_blockhash`
   - Calls `FlushSnapshotToDisk()` to persist

---

## 5. UTXO Hash Validation

### File: `src/validation.cpp`

#### `ComputeUTXOStats()` and Hash Comparison
After loading, the snapshot is validated by computing its hash.

```cpp
maybe_stats = ComputeUTXOStats(
    CoinStatsHashType::HASH_SERIALIZED,
    snapshot_coinsdb,
    m_blockman,
    [&interrupt = m_interrupt] { SnapshotUTXOHashBreakpoint(interrupt); });

// Assert hash matches expected
if (AssumeutxoHash{maybe_stats->hashSerialized} != au_data.hash_serialized) {
    return util::Error{...};  // Hash mismatch!
}
```

**What this validates:**
- All coins deserialized correctly
- No data corruption
- Snapshot matches expected UTXO set for that block height

---

## 6. Snapshot Chainstate Initialization

### File: `src/validation.cpp`

#### Loading existing snapshot on startup (line 6134)

```cpp
Chainstate* ChainstateManager::LoadAssumeutxoChainstate()
```

**What it does:**
1. Calls `node::FindAssumeutxoChainstateDir()` to find snapshot directory
2. Calls `node::ReadSnapshotBaseBlockhash()` to get base block hash
3. Creates new `Chainstate` with `m_from_snapshot_blockhash` set
4. Calls `AddChainstate()` to switch to snapshot chainstate
5. Returns pointer to new snapshot chainstate

---

### File: `src/node/chainstate.cpp`

#### In `LoadChainstate()` (line 178)
```cpp
Chainstate* assumeutxo_cs{chainman.LoadAssumeutxoChainstate()};
```

If snapshot exists:
- Snapshot chainstate becomes the "active" chainstate
- Original chainstate becomes "background" chainstate for IBD
- Both are stored in `m_chainstates` vector

---

## 7. Block Index for Snapshot

### File: `src/validation.cpp`

After snapshot activation, block index is updated:

```cpp
for (int i = AFTER_GENESIS_START; i <= snapshot_chainstate.m_chain.Height(); ++i) {
    index = snapshot_chainstate.m_chain[i];
    
    // Fake BLOCK_OPT_WITNESS so NeedsRedownload() won't ask for reindex
    if (DeploymentActiveAt(*index, *this, Consensus::DEPLOYMENT_SEGWIT)) {
        index->nStatus |= BLOCK_OPT_WITNESS;
    }
    
    m_blockman.m_dirty_blockindex.insert(index);
}
```

**What it does:**
- Sets `BLOCK_OPT_WITNESS` flag for segwit-active blocks
- Marks indices dirty for later persistence
- Sets `m_chain_tx_count` from assumeutxo data

---

## 8. Background Validation

### File: `src/validation.cpp`

#### `MaybeValidateSnapshot()` (line 5924+)
Validates snapshot in background after IBD completes.

```cpp
SnapshotCompletionResult ChainstateManager::MaybeValidateSnapshot(
    Chainstate& validated_cs,
    Chainstate& unvalidated_cs)
```

**What it does:**
1. Called after background chainstate finishes IBD
2. Compares background chainstate's UTXO hash against snapshot
3. If valid:
   - Calls `ValidatedSnapshotCleanup()` to remove background chainstate
   - Makes snapshot chainstate the sole chainstate
4. If invalid:
   - Marks snapshot as invalid
   - Keeps both chainstates for forensic analysis

---

## Summary Flow Diagram

```
User calls loadtxoutset RPC
         │
         ▼
┌─────────────────────────────────────┐
│  RPC: loadtxoutset()                │
│  - Opens file                       │
│  - Reads SnapshotMetadata           │
└─────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  ActivateSnapshot()                 │
│  - Validates base block hash        │
│  - Checks block index exists        │
│  - Resizes caches                   │
└─────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  Create snapshot Chainstate         │
│  - AddChainstate()                  │
└─────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  PopulateAndValidateSnapshot()      │
│  - Deserialize coins from file      │
│  - Validate coin data               │
│  - Flush to LevelDB periodically    │
└─────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  ComputeUTXOStats()                 │
│  - Hash UTXO set                    │
│  - Compare against expected hash    │
└─────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  Update block index                 │
│  - Set BLOCK_OPT_WITNESS flags      │
│  - Set m_chain_tx_count             │
└─────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  MaybeRebalanceCaches()             │
│  - Resize caches for normal ops     │
└─────────────────────────────────────┘
         │
         ▼
Node now synced to network tip
         │
    ┌────┴────┐
    ▼         ▼
Background    Snapshot
IBD chain     validated
completes     fully
    │             │
    └──────┬──────┘
           ▼
  MaybeValidateSnapshot()
  (background validation)
```

---

## Key Functions Summary

| Function | File | Purpose |
|----------|------|---------|
| `loadtxoutset` RPC | `src/rpc/blockchain.cpp` | User-facing snapshot loading |
| `ActivateSnapshot()` | `src/validation.cpp` | Validates and activates snapshot |
| `PopulateAndValidateSnapshot()` | `src/validation.cpp` | Deserializes UTXO set |
| `LoadAssumeutxoChainstate()` | `src/validation.cpp` | Loads existing snapshot on startup |
| `FlushSnapshotToDisk()` | `src/validation.cpp` | Persists snapshot to LevelDB |
| `MaybeValidateSnapshot()` | `src/validation.cpp` | Background validation after IBD |
| `ValidatedSnapshotCleanup()` | `src/validation.cpp` | Cleanup after successful validation |

---

## Supported Snapshot Heights

Snapshots are only valid for specific block heights that are hardcoded in chainparams. These are heights where:
1. A UTXO snapshot hash is published in the code
2. The block is well-established and unlikely to reorganize

Users must obtain snapshots from trusted sources that match a supported height. The hash validation ensures any tampering is detected.

---

## Storage

- **Snapshot chainstate directory:** `chainstate_snapshot/`
- **Base blockhash file:** `chainstate_snapshot/base_blockhash`
- **Coins database:** `chainstate_snapshot/`

On subsequent startups, `LoadAssumeutxoChainstate()` detects this directory and loads the snapshot automatically.
