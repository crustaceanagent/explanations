# Bitcoin Core Initialization: Block Index and Chainstate Loading

When bitcoind starts, it must load the block index from disk and initialize the chainstate. This document details every function called, data read, and validation performed during this process.

---

## 1. Startup Entry Point

### File: `src/init.cpp`

#### `InitAndLoadChainstate()` (line 1301)
Main entry point for chainstate and block index initialization.

```cpp
static ChainstateLoadResult InitAndLoadChainstate(
    NodeContext& node,
    bool do_reindex,
    const bool do_reindex_chainstate,
    const kernel::CacheSizes& cache_sizes,
    const ArgsManager& args)
```

**What it does:**
1. Resets any existing chainstate/mempool state
2. Applies command-line options to `ChainstateManager` and `BlockManager`
3. Creates `ChainstateManager` (opens block tree DB)
4. Calls `LoadChainstate()` to initialize chainstates
5. Calls `VerifyLoadedChainstate()` to verify blocks

**Key initialization:**
- Opens block tree database at `blocks/index/`
- Sets up mempool with consistency check ratio
- Configures cache sizes for coins DB and UTXO cache

---

## 2. Chainstate Loading

### File: `src/node/chainstate.cpp`

#### `LoadChainstate()` (line 151)
Loads and initializes chainstates.

```cpp
ChainstateLoadResult LoadChainstate(ChainstateManager& chainman, const CacheSizes& cache_sizes,
                                    const ChainstateLoadOptions& options)
```

**What it does:**
1. Logs assumed-valid block configuration
2. Sets minimum chain work threshold
3. Initializes chainstate manager caches (`m_total_coinstip_cache`, `m_total_coinsdb_cache`)
4. Calls `ChainstateManager::InitializeChainstate()` for validated chainstate
5. Calls `ChainstateManager::LoadAssumeutxoChainstate()` if loading snapshot
6. Calls `CompleteChainstateInitialization()` to finish setup

---

## 3. Complete Chainstate Initialization

### File: `src/node/chainstate.cpp`

#### `CompleteChainstateInitialization()` (line 34)
Performs the core initialization work.

```cpp
static ChainstateLoadResult CompleteChainstateInitialization(
    ChainstateManager& chainman,
    const ChainstateLoadOptions& options)
```

**What it does:**
1. **Calls `ChainstateManager::LoadBlockIndex()`** - loads block index from DB
2. Validates genesis block exists
3. Checks prune mode consistency
4. Calls `Chainstate::LoadGenesisBlock()` if needed
5. For each chainstate:
   - Calls `InitCoinsDB()` - opens coins database
   - Checks database format version
   - Calls `Chainstate::ReplayBlocks()` - recovers from interrupted flush
   - Calls `InitCoinsCache()` - allocates UTXO cache
   - Calls `Chainstate::LoadChainTip()` - sets active chain tip
6. Calls `PopulateBlockIndexCandidates()` - builds set of valid chain tips

---

## 4. Block Index Loading

### File: `src/validation.cpp`

#### `ChainstateManager::LoadBlockIndex()` (line 4896)
Loads the block index from the block tree database.

```cpp
bool ChainstateManager::LoadBlockIndex()
```

**What it does:**
1. Checks if block files are indexed (`m_blockfiles_indexed`)
2. Calls `BlockManager::LoadBlockIndexDB()`
3. Updates `m_best_header` to block with most work
4. Tracks `m_best_invalid` block (failed validation but has work)

---

### File: `src/node/blockstorage.cpp`

#### `BlockManager::LoadBlockIndexDB()` (line 529)
Loads block index from LevelDB.

```cpp
bool BlockManager::LoadBlockIndexDB(const std::optional<uint256>& snapshot_blockhash)
```

**What it does:**
1. Calls `LoadBlockIndex()` to load index entries
2. Handles assumeutxo snapshot if applicable

---

### File: `src/node/blockstorage.cpp`

#### `BlockManager::LoadBlockIndex()` (line 423)
Core block index loading.

```cpp
bool BlockManager::LoadBlockIndex(const std::optional<uint256>& snapshot_blockhash)
```

**What it does:**
1. Calls `BlockTreeDB::LoadBlockIndexGuts()` - iterates DB records
2. For each block index entry:
   - Creates `CBlockIndex` in memory
   - Links to parent (`pprev`)
   - Calculates `nChainWork` = `pprev->nChainWork + GetBlockProof(*pindex)`
   - Calculates `nTimeMax` = max block time in chain
   - Sets `m_chain_tx_count` (transaction count in chain)
   - Handles `BLOCK_FAILED_CHILD` deprecation
   - Propagates `BLOCK_FAILED_VALID` to descendants
   - Calls `BuildSkip()` for skip list

---

### File: `src/node/blockstorage.cpp`

#### `BlockTreeDB::LoadBlockIndexGuts()` (line 120)
Reads block index from LevelDB.

```cpp
bool BlockTreeDB::LoadBlockIndexGuts(const Consensus::Params& consensusParams, 
    std::function<CBlockIndex*(const uint256&)> insertBlockIndex, 
    const util::SignalInterrupt& interrupt)
```

**Data read from database:**
- `hashPrev` - parent block hash
- `nHeight` - block height
- `nFile`, `nDataPos`, `nUndoPos` - file locations
- `nVersion` - block version
- `hashMerkleRoot` - merkle root
- `nTime` - block timestamp
- `nBits` - difficulty target
- `nNonce` - PoW nonce
- `nStatus` - block status flags
- `nTx` - transaction count

**Validation performed:**
- **Proof of Work:** Calls `CheckProofOfWork(hash, nBits, consensusParams)`

---

## 5. Genesis Block Initialization

### File: `src/validation.cpp`

#### `Chainstate::LoadGenesisBlock()` (line 4924)
Loads or creates the genesis block.

```cpp
bool Chainstate::LoadGenesisBlock()
```

**What it does:**
1. Checks if genesis block already exists in `m_block_index`
2. If not found: writes genesis block to disk via `BlockManager::WriteBlock()`
3. Calls `BlockManager::AddToBlockIndex()` to add genesis to index
4. Calls `ChainstateManager::ReceivedBlockTransactions()` to mark transactions

---

### File: `src/node/blockstorage.cpp`

#### `BlockManager::AddToBlockIndex()` (line 224)
Adds a block to the in-memory index.

```cpp
CBlockIndex* BlockManager::AddToBlockIndex(const CBlockHeader& block, CBlockIndex*& best_header)
```

**What it does:**
1. Creates `CBlockIndex` entry in `m_block_index` map
2. Sets `nSequenceId = SEQ_ID_INIT_FROM_DISK`
3. Looks up parent block and links `pprev`
4. Sets `nHeight = pprev->nHeight + 1`
5. Calls `BuildSkip()` to build skip list (for reverse iteration)
6. Calculates `nTimeMax` (max time in chain)
7. Calculates `nChainWork` = `pprev->nChainWork + GetBlockProof(*pindex)`
8. Sets validity to `BLOCK_VALID_TREE`
9. Updates `best_header` if this block has more work
10. Marks index entry as dirty for later persistence

---

## 6. Coins Database Initialization

### File: `src/node/chainstate.cpp`

#### `Chainstate::InitCoinsDB()` (line ~95)
Opens the coins (UTXO) database.

```cpp
chainstate->InitCoinsDB(cache_size_bytes, in_memory, should_wipe);
```

**What it does:**
1. Opens LevelDB at `chainstate/`
2. Allocates cache of specified size
3. Checks database format version (`NeedsUpgrade()`)

---

### File: `src/node/chainstate.cpp`

#### `Chainstate::ReplayBlocks()` (line 4769)
Recovers from interrupted database flush.

```cpp
bool Chainstate::ReplayBlocks()
```

**What it does:**
1. Reads coins DB head blocks via `db.GetHeadBlocks()`
2. If heads indicate interrupted flush:
   - Finds fork point between old and new tip
   - Disconnects blocks along old chain via `DisconnectBlock()`
   - Re-applies blocks along new chain
3. Ensures UTXO set matches best block

---

### File: `src/node/chainstate.cpp`

#### `Chainstate::LoadChainTip()` (line 4542)
Initializes the active chain from the coins database.

```cpp
bool Chainstate::LoadChainTip()
```

**What it does:**
1. Reads best block hash from coins cache (`CoinsTip().GetBestBlock()`)
2. Looks up `CBlockIndex` for best block via `LookupBlockIndex()`
3. Sets chain tip via `m_chain.SetTip(*pindex)`
4. Calls `UpdateIBDStatus()` - may exit IBD
5. Sets `nSequenceId = SEQ_ID_BEST_CHAIN_FROM_DISK` for all blocks in active chain
6. Logs chain info (hash, height, date, verification progress)

---

## 7. Block Index Candidates

### File: `src/validation.cpp`

#### `Chainstate::PopulateBlockIndexCandidates()` 
Builds set of valid block index candidates for chain selection.

**What it does:**
1. Iterates all blocks in `m_block_index`
2. Adds valid blocks to `setBlockIndexCandidates`
3. Blocks must have `BLOCK_VALID_TREE` (or better) validity
4. Sorted by `CBlockIndexWorkComparator` (most work first)

---

## 8. Block Verification on Startup

### File: `src/node/chainstate.cpp`

#### `VerifyLoadedChainstate()` (line 240)
Verifies loaded blocks after initialization.

```cpp
ChainstateLoadResult VerifyLoadedChainstate(ChainstateManager& chainman, 
                                             const ChainstateLoadOptions& options)
```

**What it does:**
1. Checks for future-dated blocks (computer clock error)
2. Calls `CVerifyDB::VerifyDB()` for block verification

---

### File: `src/validation.cpp`

#### `CVerifyDB::VerifyDB()` (line 4607)
Verifies block chain integrity.

```cpp
VerifyDBResult CVerifyDB::VerifyDB(
    Chainstate& chainstate,
    const Consensus::Params& consensus_params,
    CCoinsView& coinsview,
    int nCheckLevel, int nCheckDepth)
```

**Verification levels:**

**Level 0:** Read block from disk
- Reads each block from `blk*.dat` files
- Returns error if block can't be read

**Level 1:** Verify block validity
- Calls `CheckBlock()` - all context-free checks
- Header validation, merkle root, transactions, size limits

**Level 2:** Verify undo data
- Reads undo data from `rev*.dat` files
- Ensures undo data exists for each block

**Level 3:** Disconnect and reconnect
- Disconnects tip block from chain
- Validates UTXO set changes are correct
- Reconnects block and verifies UTXO matches
- Most thorough but requires sufficient cache

**What gets validated:**
- All blocks from tip back to `nCheckDepth`
- Block can be read from disk
- `CheckBlock()` passes all tests
- Undo data exists and is valid
- (Level 3) Disconnect produces valid UTXO state
- No coin database inconsistencies

---

## Summary: What Data is Read

### From Block Index Database (`blocks/index/`):
| Field | Type | Description |
|-------|------|-------------|
| `hashPrev` | uint256 | Parent block hash |
| `nHeight` | int | Block height |
| `nFile` | int | Block file number |
| `nDataPos` | int | Position in block file |
| `nUndoPos` | int | Position in undo file |
| `nVersion` | int32_t | Block version |
| `hashMerkleRoot` | uint256 | Merkle root |
| `nTime` | uint32_t | Block timestamp |
| `nBits` | uint32_t | Difficulty target |
| `nNonce` | uint32_t | PoW nonce |
| `nStatus` | int | Block status flags |
| `nTx` | int | Transaction count |

### From Coins Database (`chainstate/`):
- Best block hash (`"best"`)
- UTXO set (all unspent transaction outputs)
- Database head blocks for replay detection

### From Block Files (`blk*.dat`):
- Full block data (transactions, witnesses)

---

## Summary: Validation Performed

1. **Proof of Work:** `CheckProofOfWork()` validates hash meets difficulty target
2. **Block index continuity:** Ensures no gaps in block heights
3. **Genesis block:** Validates or creates genesis block
4. **Prune mode:** Ensures consistency between config and on-disk state
5. **Database format:** Checks version compatibility
6. **Future time:** Detects incorrect system clock
7. **Block verification (optional):** `CheckBlock()` on recent blocks

---

## Initialization Flow Diagram

```
bitcoind starts
       │
       ▼
┌─────────────────────────────────────┐
│  init.cpp                           │
│  InitAndLoadChainstate()            │
└─────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│  node/chainstate.cpp                │
│  LoadChainstate()                   │
└─────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│  CompleteChainstateInitialization() │
└─────────────────────────────────────┘
       │
       ├──────────────────────────────┤
       ▼                              ▼
LoadBlockIndex()               InitCoinsDB()
       │                              │
       ▼                              ▼
LoadBlockIndexDB()            Check DB version
       │                              │
       ▼                              ▼
LoadBlockIndexGuts()          ReplayBlocks()
(reads from LevelDB)          (recovery if needed)
       │
       ├──────────────────────────────┤
       ▼                              ▼
AddToBlockIndex()            LoadChainTip()
(builds in-memory index)     (sets chain tip)
       │
       ▼
PopulateBlockIndexCandidates
       │
       ▼
┌─────────────────────────────────────┐
│  VerifyLoadedChainstate()           │
│  - Check block data readable        │
│  - CheckBlock() validation          │
│  - Undo data validation             │
│  - Disconnect/reconnect (level 3)   │
└─────────────────────────────────────┘
       │
       ▼
bitcoind ready for operations
```

---

## Key Files Involved

| File | Purpose |
|------|---------|
| `src/init.cpp` | Startup orchestration |
| `src/node/chainstate.cpp` | Chainstate loading logic |
| `src/node/blockstorage.cpp` | Block index storage/loading |
| `src/validation.cpp` | Block and chain validation |
| `src/kernel/coinsDB.cpp` | Coins database operations |
| `src/kernel/blocktreeDB.cpp` | Block index database |

---

## CBlockIndex Data Structure

The in-memory block index (`CBlockIndex`) stores:

```cpp
struct CBlockIndex {
    const uint256* phashBlock;     // Pointer to block hash
    CBlockIndex* pprev;            // Parent block
    CBlockIndex* pskip;            // Skip list entry (for reverse iteration)
    int nHeight;                   // Block height
    int nFile;                     // Block file number
    int nDataPos;                  // Position in block file
    int nUndoPos;                  // Position in undo file
    int32_t nVersion;              // Block version
    uint256 hashMerkleRoot;        // Merkle root
    uint32_t nTime;                // Block timestamp
    uint32_t nBits;                // Difficulty target
    uint32_t nNonce;               // PoW nonce
    uint64_t nChainWork;           // Cumulative work
    uint64_t nTx;                  // Transaction count
    int64_t m_chain_tx_count;      // Tx count in chain up to this block
    int64_t nTimeMax;              // Max time in chain
    int nStatus;                   // Status flags
    int nSequenceId;               // Sequence for candidate selection
    // ... more fields
};
```

This structure is built from database records and used for chain navigation, validation, and block lookup throughout bitcoind's operation.
