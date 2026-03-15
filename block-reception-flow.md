# Bitcoin Block Reception and Validation Flow

When Bitcoin Core receives a block from a peer, it goes through a complex multi-stage validation and processing pipeline. This document details every function call from networking to disk write.

---

## 1. Networking Layer

### File: `src/net_processing.cpp`

#### `ProcessBlock()` (line 3421)
Entry point for processing a newly received block from a peer.

```cpp
void PeerManagerImpl::ProcessBlock(CNode& node, const std::shared_ptr<const CBlock>& block, bool force_processing, bool min_pow_checked)
```

**What it does:**
- Calls `ChainstateManager::ProcessNewBlock()` to validate and accept the block
- Tracks whether the block is new via the `new_block` output parameter
- Updates peer's last block time
- Removes block from request tracking

---

## 2. Block Validation - Main Entry

### File: `src/validation.cpp`

#### `ChainstateManager::ProcessNewBlock()` (line 4394)
Main entry point for processing a new block.

```cpp
bool ChainstateManager::ProcessNewBlock(const std::shared_ptr<const CBlock>& block, bool force_processing, bool min_pow_checked, bool* new_block)
```

**What it does:**
1. Calls `CheckBlock()` - context-free validation
2. Calls `AcceptBlock()` - stores block to disk and updates index
3. Calls `ActivateBestChain()` - attempts to make this block the new tip
4. Calls background chainstate's `ActivateBestChain()` if exists

---

## 3. Context-Free Block Validation

### File: `src/validation.cpp`

#### `CheckBlock()` (line 3914)
Performs context-free checks that don't depend on chain state.

```cpp
bool CheckBlock(const CBlock& block, BlockValidationState& state, const Consensus::Params& consensusParams, bool fCheckPOW, bool fCheckMerkleRoot)
```

**Checks performed:**
1. `CheckBlockHeader()` - validates PoW and header integrity
2. `CheckSignetBlockSolution()` - signet signature validation (if applicable)
3. `CheckMerkleRoot()` - merkle root verification
4. **Size limits:**
   - Block must not be empty
   - `vtx.size() * WITNESS_SCALE_FACTOR <= MAX_BLOCK_WEIGHT`
   - `TX_NO_WITNESS(block) * WITNESS_SCALE_FACTOR <= MAX_BLOCK_WEIGHT`
5. **Coinbase validation:**
   - First transaction must be coinbase
   - No other transactions can be coinbase
6. **Transaction validation:**
   - Calls `CheckTransaction()` for each tx (see section 7)
   - Checks for duplicate inputs (CVE-2018-17144)
7. **SigOps limit:**
   - `nSigOps * WITNESS_SCALE_FACTOR <= MAX_BLOCK_SIGOPS_COST`

---

### File: `src/validation.cpp`

#### `CheckBlockHeader()` (line 3824)
Validates the block header itself.

```cpp
static bool CheckBlockHeader(const CBlockHeader& block, BlockValidationState& state, const Consensus::Params& consensusParams, bool fCheckPOW = true)
```

**Checks:**
- **Proof of Work:** `CheckProofOfWork(block.GetHash(), block.nBits, consensusParams)`

---

### File: `src/validation.cpp`

#### `CheckMerkleRoot()` (line 3837)
Verifies the merkle root in the header matches computed merkle root.

```cpp
static bool CheckMerkleRoot(const CBlock& block, BlockValidationState& state)
```

---

## 4. Block Header Acceptance

### File: `src/validation.cpp`

#### `ChainstateManager::AcceptBlockHeader()` (line 4182)
Accepts a block header into the block index (headers processing).

```cpp
bool ChainstateManager::AcceptBlockHeader(const CBlockHeader& block, BlockValidationState& state, CBlockIndex** ppindex, bool min_pow_checked)
```

**What it does:**
1. Checks for duplicate headers in `m_block_index`
2. Calls `CheckBlockHeader()` (validates PoW)
3. Checks proof-of-work against anti-DoS thresholds
4. `ContextualCheckBlockHeader()` - checks timestamp, chain work, deployment state
5. Creates/updates `CBlockIndex` entry in memory
6. Updates `m_best_header` if this header has more work

---

### File: `src/validation.cpp`

#### `ContextualCheckBlockHeader()` (line 4076)
Context-dependent header validation.

```cpp
static bool ContextualCheckBlockHeader(const CBlockHeader& block, BlockValidationState& state, BlockManager& blockman, const ChainstateManager& chainman, const CBlockIndex* pindexPrev)
```

**Checks:**
- Block isn't too far in the future (2-hour limit)
- Chain work is at least minimum required
- Checkpoints (if enabled)
- Validates deployment state (taproot, segwit, etc.)

---

## 5. Block Body Acceptance

### File: `src/validation.cpp`

#### `ChainstateManager::AcceptBlock()` (line 4294)
Accepts a full block (header + body) into the block index.

```cpp
bool ChainstateManager::AcceptBlock(const std::shared_ptr<const CBlock>& pblock, BlockValidationState& state, CBlockIndex** ppindex, bool fRequested, const FlatFilePos* dbp, bool* fNewBlock, bool min_pow_checked)
```

**What it does:**
1. Calls `AcceptBlockHeader()` - validates header and adds to index
2. Checks if block already has data (`BLOCK_HAVE_DATA`)
3. Anti-DoS checks for unrequested blocks:
   - Must have more work than current tip
   - Not too far ahead of current height
   - Must meet minimum chain work threshold
4. Calls `CheckBlock()` again with full context
5. Calls `ContextualCheckBlock()` for context-dependent validation
6. **Writes block to disk** (see section 8)
7. Calls `ReceivedBlockTransactions()`
8. Calls `FlushStateToDisk()`

---

### File: `src/validation.cpp`

#### `ContextualCheckBlock()` (line 4125)
Context-dependent block validation.

```cpp
static bool ContextualCheckBlock(const CBlock& block, BlockValidationState& state, const ChainstateManager& chainman, const CBlockIndex* pindexPrev)
```

**Checks:**
1. **BIP113 (Median Time Past):** Enforce locktime using MTP
2. **Transaction finality:** All transactions must be final
3. **BIP34 (Height in Coinbase):** Coinbase must contain block height
4. **Witness malleation:** Check witness commitment structure
5. **Block weight:** `GetBlockWeight(block) <= MAX_BLOCK_WEIGHT`

---

## 6. Transaction Validation

### File: `src/consensus/tx_check.cpp`

#### `CheckTransaction()` (line 11)
Basic transaction validation independent of context.

```cpp
bool CheckTransaction(const CTransaction& tx, TxValidationState& state)
```

**Checks:**
- `vin` not empty
- `vout` not empty
- Size limits (without witness)
- No negative or overflow output values
- No duplicate inputs
- Coinbase scriptSig size (2-100 bytes)
- Non-coinbase transactions have valid prevouts

---

### File: `src/consensus/tx_verify.cpp`

#### `Consensus::CheckTxInputs()` (line 164)
Validates transaction inputs against UTXO set.

```cpp
bool Consensus::CheckTxInputs(const CTransaction& tx, TxValidationState& state, const CCoinsViewCache& inputs, int nSpendHeight, CAmount& txfee)
```

**Checks:**
- All inputs exist in UTXO set
- No spent inputs (double-spend check)
- ScriptSig satisfies scriptPubKey (via `CheckInputScripts`)
- Total input value >= total output value (fee calculation)

---

### File: `src/validation.cpp`

#### `CheckInputScripts()` (line 2058)
Script validation for transaction inputs.

```cpp
bool CheckInputScripts(const CTransaction& tx, TxValidationState& state, const CCoinsViewCache& inputs, script_verify_flags flags, bool cacheSigStore, bool cacheFullScriptStore, PrecomputedTransactionData& txdata, ValidationCache& validation_cache, std::vector<CScriptCheck>* pvChecks)
```

**What it does:**
- Validates ECDSA signatures
- Validates script execution
- Supports P2SH, P2WPKH, P2WSH, P2TR
- Uses script execution cache for performance

---

### File: `src/validation.cpp`

#### `GetBlockScriptFlags()` (line 2247)
Determines script verification flags based on deployment state.

```cpp
script_verify_flags GetBlockScriptFlags(const CBlockIndex& block_index, const ChainstateManager& chainman)
```

**Returns flags based on active soft forks:**
- `SCRIPT_VERIFY_P2SH` - BIP16 P2SH
- `SCRIPT_VERIFY_WITNESS` - BIP141 SegWit
- `SCRIPT_VERIFY_TAPROOT` - BIP341 Taproot
- `SCRIPT_VERIFY_DERSIG` - BIP66 DERSIG
- `SCRIPT_VERIFY_CHECKLOCKTIMEVERIFY` - BIP65 CLTV
- `SCRIPT_VERIFY_CHECKSEQUENCEVERIFY` - BIP112 CSV
- `SCRIPT_VERIFY_NULLDUMMY` - BIP147

---

## 7. Block Connection to Chain

### File: `src/validation.cpp`

#### `Chainstate::ActivateBestChain()` (line 3319)
Attempts to make the new block (or chain) the active tip.

```cpp
bool Chainstate::ActivateBestChain(BlockValidationState& state, std::shared_ptr<const CBlock> pblock)
```

**What it does:**
1. Calls `FindMostWorkChain()` to find best candidate
2. Calls `ActivateBestChainStep()` to connect blocks
3. Disconnects blocks no longer in best chain via `DisconnectTip()`
4. Connects new blocks via `ConnectTip()`
5. Sends `BlockConnected` signals
6. Notifies `UpdatedBlockTip` listeners

---

### File: `src/validation.cpp`

#### `Chainstate::ActivateBestChainStep()` (line 3187)
Processes one step of chain activation.

```cpp
bool Chainstate::ActivateBestChainStep(BlockValidationState& state, CBlockIndex* pindexMostWork, const std::shared_ptr<const CBlock>& pblock, bool& fInvalidFound, std::vector<ConnectedBlock>& connected_blocks)
```

**What it does:**
1. Finds fork point with current tip
2. **Disconnects** blocks from old tip via `DisconnectTip()`
3. **Connects** blocks to new tip via `ConnectTip()`
4. Handles chain reorganizations
5. Prunes block index candidates

---

### File: `src/validation.cpp`

#### `Chainstate::ConnectTip()` (line 3001)
Connects a block to the current tip.

```cpp
bool Chainstate::ConnectTip(BlockValidationState& state, CBlockIndex* pindexNew, std::shared_ptr<const CBlock> block_to_connect, std::vector<ConnectedBlock>& connected_blocks, DisconnectedBlockTransactions& disconnectpool)
```

**What it does:**
1. Loads block from disk if not provided
2. Creates new coins view for block connection
3. Calls `ConnectBlock()`
4. Flushes coins view to cache
5. Calls `FlushStateToDisk()` if needed

---

### File: `src/validation.cpp`

#### `Chainstate::ConnectBlock()` (line 2292)
Core block connection - validates and applies block to UTXO.

```cpp
bool Chainstate::ConnectBlock(const CBlock& block, BlockValidationState& state, CBlockIndex* pindex, CCoinsViewCache& view, bool fJustCheck)
```

**What it does:**
1. Calls `CheckBlock()` again (safety check)
2. **BIP30 check:** No duplicate transactions
3. **BIP34 check:** Height in coinbase (if applicable)
4. **BIP68:** Sequence lock checks
5. Gets script flags via `GetBlockScriptFlags()`
6. **For each transaction:**
   - Calls `Consensus::CheckTxInputs()` - validates inputs exist and have enough value
   - Calls `CheckInputScripts()` - validates scripts
   - Tracks sigop cost
7. Validates coinbase value doesn't exceed block subsidy + fees
8. **Writes undo data** for potential reorgs via `WriteBlockUndo()`
9. Updates block index validity to `BLOCK_VALID_SCRIPTS`
10. Calls `view.SetBestBlock()` - updates UTXO tip

---

### File: `src/validation.cpp`

#### `UpdateCoins()` (line 1996)
Updates UTXO set with block transactions.

```cpp
void UpdateCoins(const CTransaction& tx, CCoinsViewCache& inputs, CTxUndo &txundo, int nHeight)
```

**What it does:**
1. For non-coinbase txs: marks inputs as spent (via `SpendCoin()`)
2. Adds new UTXOs via `AddCoins()`

---

## 8. Block Storage (Writing to Disk)

### File: `src/node/blockstorage.cpp`

#### `BlockManager::WriteBlock()` (line 1134)
Writes block to disk in blkNNNNN.dat file.

```cpp
FlatFilePos BlockManager::WriteBlock(const CBlock& block, int nHeight)
```

**What it does:**
1. Calculates block size
2. Calls `FindNextBlockPos()` to find position in block file
3. Calls `OpenBlockFile()` to open file
4. Writes block file header (magic bytes + size)
5. Writes serialized block with witness data (`TX_WITH_WITNESS`)
6. Closes file

**Files written to:** `blocks/blkNNNNN.dat`

---

### File: `src/node/blockstorage.cpp`

#### `BlockManager::WriteBlockUndo()` (line 967)
Writes undo data for reversing block connection.

```cpp
bool BlockManager::WriteBlockUndo(const CBlockUndo& blockundo, BlockValidationState& state, CBlockIndex& block)
```

**What it does:**
- Writes to `blocks/revNNNNN.dat`
- Stores previous UTXO state for reorg capability

---

## 9. Disconnect (Chain Reorganization)

### File: `src/validation.cpp`

#### `Chainstate::DisconnectTip()` 
Disconnects the current tip from the chain.

**What it does:**
1. Removes block from chain
2. Reverses UTXO changes (spends become available, outputs are removed)
3. Returns transactions to mempool

---

## Summary Flow Diagram

```
Peer sends BLOCK message
         │
         ▼
┌─────────────────────────────────────┐
│  net_processing.cpp                 │
│  ProcessBlock()                     │
└─────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  validation.cpp                     │
│  ChainstateManager::ProcessNewBlock │
└─────────────────────────────────────┘
         │
    ┌────┴────┐
    ▼         ▼
CheckBlock()  AcceptBlockHeader()
    │              │
    │         ContextualCheckBlockHeader()
    │              │
    │         Creates CBlockIndex
    ▼              ▼
AcceptBlock() ◄────────────┘
    │
    ├─► ContextualCheckBlock()
    │
    ├─► BlockManager::WriteBlock() ──► blkNNNNN.dat
    │
    ├─► ReceivedBlockTransactions()
    │
    └─► ActivateBestChain()
              │
              ├─► DisconnectTip() (if reorg)
              │
              └─► ConnectTip()
                    │
                    └─► ConnectBlock()
                          │
                          ├─► CheckTransaction()
                          ├─► Consensus::CheckTxInputs()
                          ├─► CheckInputScripts()
                          ├─► UpdateCoins()
                          ├─► WriteBlockUndo() ──► revNNNNN.dat
                          └─► view.SetBestBlock()
```

---

## Key Files Involved

| File | Purpose |
|------|---------|
| `src/net_processing.cpp` | P2P message handling |
| `src/validation.cpp` | Block and transaction validation |
| `src/node/blockstorage.cpp` | Block disk I/O |
| `src/consensus/tx_check.cpp` | Basic tx validation |
| `src/consensus/tx_verify.cpp` | Input validation |
| `src/script/interpreter.cpp` | Script execution |

---

## Validation Cache

Bitcoin Core uses two caches during validation:

1. **Script Execution Cache:** Caches script verification results to avoid re-running ECDSA checks
2. **Signature Cache:** Caches signature validation results

These significantly speed up block validation when blocks contain previously-seen transactions.
