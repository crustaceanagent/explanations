# Bitcoin Core Package Processing: Parent-Child Transaction Handling

This document explains how Bitcoin Core processes packages of dependent transactions—specifically parent-child packages, larger bundles, and how the P2P network handles them. It assumes you already understand single transaction broadcast and RBF (Replace-By-Fee).

---

## 1. Package Concepts

### What is a Package?

A **package** is a collection of transactions that depend on each other. Bitcoin Core supports:

1. **Single transaction** - Already understood (standard broadcast)
2. **1-parent-1-child (1P1C)** - One parent spends outputs, one child spends parent's outputs
3. **Child-with-parents** - One child with multiple parents
4. **Larger packages** - Multiple dependent transactions (limited to MAX_PACKAGE_COUNT)

### Package Topology Rules

### File: `src/policy/packages.cpp`

#### `IsChildWithParents()` (line 119)
Validates that a package is a "child-with-parents" topology:

```cpp
bool IsChildWithParents(const Package& package)
```

**Requirements:**
- Package has ≥2 transactions
- Last transaction is the **child** (spends from parents)
- All other transactions are **parents** of the child
- Parents must all be inputs to the child

#### `IsWellFormedPackage()` (line 79)
Validates package structure:

```cpp
bool IsWellFormedPackage(const Package& txns, PackageValidationState& state)
```

**Checks:**
- Max 25 transactions (`MAX_PACKAGE_COUNT`)
- Max 101 KB total weight (`MAX_PACKAGE_WEIGHT`)
- No duplicate transactions
- Topologically sorted (parents before children)
- No conflicting inputs within package

---

## 2. ATMPArgs: Validation Modes

### File: `src/validation.cpp`

The `ATMPArgs` struct controls how transactions are validated:

### Single Transaction (Standard)

```cpp
static ATMPArgs SingleAccept(...)
```

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `m_test_accept` | false | Actually submit to mempool |
| `m_allow_replacement` | true | Allow RBF |
| `m_allow_sibling_eviction` | true | Allow evicting siblings |
| `m_package_submission` | false | Not a package |
| `m_package_feerates` | false | Use individual feerates |

### Package Test Accept (testmempoolaccept RPC)

```cpp
static ATMPArgs PackageTestAccept(...)
```

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `m_test_accept` | true | Don't submit to mempool |
| `m_allow_replacement` | false | No RBF in package test |
| `m_package_submission` | false | Test only |
| `m_package_feerates` | false | Use individual feerates |

### Child-With-Parents Package

```cpp
static ATMPArgs PackageChildWithParents(...)
```

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `m_test_accept` | false | Submit to mempool |
| `m_allow_replacement` | true | Allow RBF |
| `m_allow_sibling_eviction` | false | No sibling eviction |
| `m_package_submission` | true | Package submission mode |
| `m_package_feerates` | true | Use package feerate for fees |

---

## 3. Single Transaction Reception

### File: `src/net_processing.cpp`

When a peer sends a `TX` message:

#### `PeerManagerImpl::ProcessMessage()` - TX handling (line 4464)

```cpp
if (msg_type == NetMsgType::TX) {
    // ... validation ...
    
    const auto& [should_validate, package_to_validate] = m_txdownloadman.ReceivedTx(pfrom.GetId(), ptx);
    
    if (!should_validate) {
        // Already have this transaction
        if (package_to_validate) {
            // Found a 1P1C package to validate!
            const auto package_result = ProcessNewPackage(...);
            ProcessPackageResult(package_to_validate.value(), package_result);
        }
        return;
    }
    
    // Validate single transaction
    const MempoolAcceptResult result = m_chainman.ProcessTransaction(ptx);
    
    if (result.m_result_type == MempoolAcceptResult::ResultType::VALID) {
        ProcessValidTx(pfrom.GetId(), ptx, result.m_replaced_transactions);
    }
    
    if (state.IsInvalid()) {
        // Try to find package with missing parents
        if (auto package_to_validate{ProcessInvalidTx(pfrom.GetId(), ptx, state, /*first_time_failure=*/true)}) {
            const auto package_result = ProcessNewPackage(...);
            ProcessPackageResult(package_to_validate.value(), package_result);
        }
    }
}
```

---

## 4. 1-Parent-1-Child (1P1C) Package Detection

### File: `src/node/txdownloadman_impl.cpp`

When a transaction fails with `TX_MISSING_INPUTS`, Bitcoin Core checks for dependent transactions in the orphanage.

#### `TxDownloadManagerImpl::Find1P1CPackage()` (line 297)

```cpp
std::optional<PackageToValidate> Find1P1CPackage(const CTransactionRef& ptx, NodeId nodeid)
```

**What it does:**
1. Looks up children of the failed transaction in the **orphanage**
2. Filters children from the **same peer** (prevents censorship attacks)
3. Returns {parent, child} package if:
   - Child not in recent rejects filter
   - Package not in reconsiderable rejects filter

#### `TxDownloadManagerImpl::MempoolRejectedTx()` (line 350)

When a transaction fails:

```cpp
RejectedTxTodo MempoolRejectedTx(...)
```

**If `TX_MISSING_INPUTS`:**
1. Adds transaction to orphanage (orphan)
2. Fetches unique parent txids
3. Checks if parents are in recent rejects
4. **Calls `Find1P1CPackage()`** to find children in orphanage
5. Returns package to validate if found

---

## 5. Package Validation Flow

### File: `src/validation.cpp`

#### `MemPoolAccept::AcceptPackage()` (line 1619)

Main entry point for package validation:

```cpp
PackageMempoolAcceptResult AcceptPackage(const Package& package, ATMPArgs& args)
```

**Step 1: Context-free validation**
```cpp
if (!IsWellFormedPackage(package, ...)) return error;
if (package.size() > 1 && !IsChildWithParents(package)) return error;
```

**Step 2: Process each transaction individually**
```cpp
for (const auto& tx : package) {
    if (tx already in mempool) {
        // Skip, use existing entry
    } else {
        // Try individually first
        single_result = AcceptSubPackage({tx}, args);
        
        if (single_result VALID) {
            // Already in mempool, done
        } else if (fails for reasons OTHER than feerate/missing inputs) {
            // Can't succeed as package, quit early
            quit_early = true;
        } else {
            // Might succeed as package, queue for package eval
            txns_package_eval.push_back(tx);
        }
    }
}
```

**Step 3: Package validation (if needed)**
```cpp
if (!quit_early && !txns_package_eval.empty()) {
    multi_result = AcceptSubPackage(txns_package_eval, args);
}
```

**Step 4: Eviction check**
```cpp
LimitMempoolSize(m_pool, ...);  // May evict entries
```

---

## 6. Subpackage Validation

### File: `src/validation.cpp`

#### `MemPoolAccept::AcceptSubPackage()` (line 1593)

```cpp
PackageMempoolAcceptResult AcceptSubPackage(const std::vector<CTransactionRef>& subpackage, ATMPArgs& args)
```

**If subpackage size == 1:**
- Calls `AcceptSingleTransactionInternal()` with adjusted args

**If subpackage size > 1:**
- Calls `AcceptMultipleTransactionsInternal()`

---

## 7. Multi-Transaction Validation

### File: `src/validation.cpp`

#### `MemPoolAccept::AcceptMultipleTransactionsInternal()` (line 1429)

```cpp
PackageMempoolAcceptResult AcceptMultipleTransactionsInternal(const std::vector<CTransactionRef>& txns, ATMPArgs& args)
```

**Step 1: PreChecks on all transactions**
```cpp
for (Workspace& ws : workspaces) {
    if (!PreChecks(args, ws)) {
        return error;  // Fail fast
    }
    
    // Check individual modified feerate
    if (args.m_client_maxfeerate && ws.m_modified_fees / ws.m_vsize > maxfeerate) {
        return error;
    }
    
    // Make coins available for subsequent txs
    m_viewmempool.PackageAddTransaction(ws.m_ptx);
}
```

**Step 2: TRUC checks**
```cpp
for (Workspace& ws : workspaces) {
    if (auto err{PackageTRUCChecks(...)}) {
        return error;
    }
}
```

**Step 3: Package feerate calculation**
```cpp
m_subpackage.m_total_vsize = sum(ws.m_vsize);
m_subpackage.m_total_modified_fees = sum(ws.m_modified_fees);
CFeeRate package_feerate(total_modified_fees, total_vsize);
```

**Step 4: Package RBF checks**
```cpp
if (m_subpackage.m_rbf && !PackageRBFChecks(txns, ...)) {
    return error;
}
```

**Step 5: Cluster limit check**
```cpp
if (!m_subpackage.m_changeset->CheckMemPoolPolicyLimits()) {
    return error;  // "too-large-cluster"
}
```

**Step 6: Dust spend check**
```cpp
if (!CheckEphemeralSpends(txns, ...)) {
    return error;  // "unspent-dust"
}
```

**Step 7: Script checks**
```cpp
for (Workspace& ws : workspaces) {
    if (!ConsensusScriptChecks(args, ws)) {
        return error;
    }
    // Immediately submit to make outputs available
    m_pool.AddUnchecked(...);
}
```

---

## 8. Package Submission

### File: `src/validation.cpp`

#### `MemPoolAccept::SubmitPackage()` (line 1239)

```cpp
bool SubmitPackage(const ATMPArgs& args, std::vector<Workspace>& workspaces, ...)
```

**What it does:**
1. Calls `FinalizeSubpackage()` - calculates fees, handles conflicts
2. For each transaction:
   - Calls `ConsensusScriptChecks()` - signature verification
   - Adds to mempool via `m_pool.AddUnchecked()`
3. Records effective feerates (individual or package-based)
4. Handles RBF replacements if any

---

## 9. Network Messages for Packages

### P2P Message Flow

#### Scenario 1: Parent with Missing Inputs

```
Peer sends TX (parent with missing inputs)
         │
         ▼
    ┌────────────┐
    │ ProcessTx  │
    └────────────┘
         │
         ▼
    TX_MISSING_INPUTS
         │
         ▼
    Add to orphanage
         │
         ▼
    Find 1P1C package?
         │                    No
         ├────────────────────► Return (no package)
         │
        Yes
         │
         ▼
    ProcessNewPackage()
         │
         ├─► AcceptPackage()
         │       │
         │       ├─► AcceptSubPackage()
         │       │       │
         │       │       └─► AcceptMultipleTransactionsInternal()
         │       │               │
         │       │               ├─► PreChecks() x2
         │       │               ├─► PackageTRUC checks
         │       │               ├─► Package feerate calc
         │       │               ├─► ConsensusScriptChecks() x2
         │       │               └─► SubmitPackage()
         │       │
         │       └─► LimitMempoolSize()
         │
         ▼
    ProcessPackageResult()
         │
    ┌────┴────┐
    │          │
 Valid     Invalid
    │          │
    ▼          ▼
Relay TX   Reject msg
to peers   to peer
```

#### Scenario 2: Package Already Known (1P1C)

```
Peer A sends TX (child)
         │
         ▼
    Already in reconsiderable rejects?
         │
        Yes (previously failed)
         │
         ▼
    Find1P1CPackage()
    (look for parent in orphanage)
         │
         ▼
    Found parent + child package!
         │
         ▼
    ProcessNewPackage()
         │
         ▼
    (same as above)
```

---

## 10. RPC Interface: testmempoolaccept

### File: `src/rpc/mempool.cpp`

The `testmempoolaccept` RPC allows testing packages without submission:

```bash
bitcoin-cli testmempoolaccept '["<raw-parent-hex>", "<raw-child-hex>"]'
```

**Requirements:**
- Parents must come before children (topologically sorted)
- Max 25 transactions
- No conflicts within package
- Uses `PackageTestAccept` mode (test only, no submission)

**Returns:**
```json
[
  {
    "txid": "...",
    "wtxid": "...",
    "allowed": true,
    "vsize": 123,
    "fees": {
      "base": "0.00010000",
      "effective-feerate": "0.00005000",
      "effective-includes": ["wtxid1", "wtxid2"]
    }
  },
  {
    "txid": "...",
    "wtxid": "...",
    "allowed": false,
    "reject-reason": "TX_MISSING_INPUTS"
  }
]
```

---

## 11. Larger Packages

### Current Limitations

Bitcoin Core currently only supports **child-with-parents** packages (all transactions except the last are parents of the last). This means:

- **1P1C**: ✅ Supported
- **Multiple parents, 1 child**: ✅ Supported  
- **Multiple generations** (grandparent → parent → child): ❌ Not as a package

For multi-generational transactions, you must submit parents individually first, then the child.

### Package Limits

| Limit | Value |
|-------|-------|
| `MAX_PACKAGE_COUNT` | 25 transactions |
| `MAX_PACKAGE_WEIGHT` | 101,000,000 weight units (~101 KB) |

---

## Summary: Key Functions

| Function | File | Purpose |
|----------|------|---------|
| `AcceptPackage()` | `validation.cpp` | Main package validation entry |
| `AcceptSubPackage()` | `validation.cpp` | Routes to single/multi validation |
| `AcceptMultipleTransactionsInternal()` | `validation.cpp` | Multi-tx validation |
| `SubmitPackage()` | `validation.cpp` | Submits validated package to mempool |
| `Find1P1CPackage()` | `txdownloadman_impl.cpp` | Finds 1P1C in orphanage |
| `MempoolRejectedTx()` | `txdownloadman_impl.cpp` | Handles missing inputs |
| `IsWellFormedPackage()` | `packages.cpp` | Validates package structure |
| `IsChildWithParents()` | `packages.cpp` | Validates topology |
| `testmempoolaccept` RPC | `mempool.cpp` | RPC package testing |

---

## Summary: Package Processing Flow

```
Received TX from peer
         │
         ▼
Has transaction? ──YES──► Skip (already have)
         │
        NO
         │
         ▼
Validate individually
         │
    ┌────┴────┐
    │         │
  VALID    INVALID
    │         │
    ▼         ▼
Relay TX   Missing inputs?
    │         │
            YES
    │         │
    ▼         ▼
    Check orphanage
    for children
    (Find1P1CPackage)
         │
    ┌────┴────┐
    │         │
  Found    Not found
    │         │
    ▼         ▼
ProcessNewPackage Accept single
    │         (no package)
    ├─► AcceptPackage()
    │     │
    │     ├─► Individual checks first
    │     │
    │     ├─► AcceptMultipleTransactionsInternal()
    │     │     │
    │     │     ├─► PreChecks() all
    │     │     ├─► TRUC checks
    │     │     ├─► Package feerate
    │     │     ├─► RBF checks
    │     │     ├─► Script checks
    │     │     └─► SubmitPackage()
    │     │
    │     └─► LimitMempoolSize()
    │
    ▼
ProcessPackageResult()
    │
    ├─► VALID: Relay to peers
    └─► INVALID: Reject to peer
```
