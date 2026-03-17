# Victionchain Legacy Transaction Workflows

This document describes the complete transaction processing pipeline in the **victionchain**
legacy client (the reference implementation). It is the authoritative source for porting
decisions to `vic-geth`.

See also: `docs/viction-tx-workflows.md` for the vic-geth equivalent and per-mechanism
differences.

---

## Block Processing Entry Point

**`victionchain/core/state_processor.go` — `Process()`**

```
Process(block, statedb, tradingState *TradingStateDB, cfg, balanceFee map[Address]*big.Int)
```

`balanceFee` is pre-populated from parent state by the caller (blockchain.go) before
`Process` is invoked:

```go
// blockchain.go (caller)
feeCapacity := state.GetTRC21FeeCapacityFromStateWithCache(parent.Root(), statedb)
receipts, logs, usedGas, err := processor.Process(block, statedb, tradingState, cfg, feeCapacity)
```

`GetTRC21FeeCapacityFromStateWithCache` reads every registered TRC21 token address from the
`TRC21IssuerSMC` contract and returns `map[tokenAddr → currentFeeCapacity]`. The map is
LRU-cached by parent state root so repeated block replays are cheap.

### Block-start state mutations

```
if block == TIPSigningBlock:
    statedb.DeleteAddress(BlockSigners)          // 0x89 deleted permanently

if block >= SaigonBlock:
    ApplySaigonHardFork(statedb, ...)

parentState := statedb.Copy()                    // snapshot for Finalize
balanceUpdated := map[Address]*big.Int{}         // tracks capacity changes this block
totalFeeUsed   := big.NewInt(0)                  // accumulates all token fees this block

InitSignerInTransactions(config, header, txs)    // parallel signer cache warm-up
```

---

## Per-Transaction Loop

```
for i, tx := range block.Transactions():

    1. Blacklist check (if block >= BlackListHFBlock = 9,349,100):
           reject if tx.From ∈ Blacklist OR tx.To ∈ Blacklist

    2. TomoZ apply-transaction validation (if IsTomoZEnabled):
           if tx.IsTomoZApplyTransaction():
               ValidateTomoZApplyTransaction(bc, statedb.Copy(), tokenAddr)

    3. TomoX apply-transaction validation (if IsTomoXEnabled):
           if tx.IsTomoXApplyTransaction():
               ValidateTomoXApplyTransaction(bc, statedb.Copy(), tokenAddr)

    4. statedb.Prepare(tx.Hash, block.Hash, i)

    5. receipt, gas, err, tokenFeeUsed = ApplyTransaction(config, balanceFee, bc,
                                                          nil, gp, statedb,
                                                          tradingState, header,
                                                          tx, &usedGas, cfg)

    6. if tokenFeeUsed && !isAtlas:
           fee = gas
           if block > TIPTRC21FeeBlock:
               fee = gas × TRC21GasPrice
           balanceFee[*tx.To()] -= fee
           balanceUpdated[*tx.To()] = balanceFee[*tx.To()]
           totalFeeUsed += fee

after loop:
    if !isAtlas:
        state.UpdateTRC21Fee(statedb, balanceUpdated, totalFeeUsed)
    engine.Finalize(bc, header, statedb, parentState, txs, uncles, receipts)
```

---

## `ApplyTransaction()` — Transaction Routing

**`victionchain/core/state_processor.go`**

All transactions pass through this function. It routes system transactions before EVM
execution and determines whether TRC21 sponsorship applies.

### System transaction shortcuts (return immediately, no gas, `tokenFeeUsed=false`)

| Condition | Handler | Address |
|---|---|---|
| `tx.To == BlockSigners` AND `IsTIPSigning` | `ApplySignTransaction()` | `0x89` |
| `tx.To == TradingStateAddr` AND `IsTIPTomoX` | `ApplyEmptyTransaction()` | `0x88` |
| `tx.IsLendingApplyTransaction()` AND `IsTIPTomoX` | `ApplyEmptyTransaction()` | — |
| `tx.IsTradingTransaction()` AND `IsTIPTomoX` | `ApplyEmptyTransaction()` | — |
| `tx.IsLendingFinalizedTradeTransaction()` AND `IsTIPTomoX` | `ApplyEmptyTransaction()` | — |

### TRC21 fee capacity lookup

```go
var balanceFee *big.Int
if tx.To() != nil {
    if value, ok := tokensFee[*tx.To()]; ok {
        balanceFee = value   // non-nil: this token is TRC21-sponsored
    }
}
```

### Message construction and gasPrice override

```go
msg = tx.AsMessage(signer, balanceFee, header.Number, true)
```

Inside `AsMessage`, if `balanceFee != nil` (TRC21 tx):

```
if block > TIPTRC21FeeBlock (13,523,400):
    msg.gasPrice = TRC21GasPrice        // standard sponsored rate
else:
    msg.gasPrice = TRC21GasPriceBefore  // pre-fork rate
```

This **overrides the user-supplied gas price** for TRC21 transactions. The user's
original gas price is ignored; the canonical fee rate is used instead.

### Coinbase owner resolution

```go
beneficiary, _ = engine.Author(header)
coinbaseOwner   = statedb.GetOwner(beneficiary)  // validator's registered owner
```

The resolved `coinbaseOwner` is passed down to `ApplyMessage → TransitionDb` for
fee routing at the TIPTRC21Fee boundary.

### Failed-tx penalty

After EVM execution:

```go
if balanceFee != nil && txFailed {
    state.PayFeeWithTRC21TxFail(statedb, msg.From(), *tx.To())
}
```

`PayFeeWithTRC21TxFail` charges `min(senderTokenBalance, minFee)` in tokens from the
sender to the token issuer. This fires only when:
- the tx is TRC21-sponsored (`balanceFee != nil`), AND
- the EVM reverted (`failed == true`)

### Return value

```go
return receipt, gasUsed, err, tokenFeeUsed   // tokenFeeUsed = (balanceFee != nil)
```

`tokenFeeUsed` propagates back to `Process()` to gate the `balanceFee` map update.

---

## `TransitionDb()` — Gas Accounting

**`victionchain/core/state_transition.go`**

```
preCheck()
  └─ nonce check
  └─ buyGas()

intrinsic gas deduction

EVM: Create or Call
  → nonce increment
  → evm.Call / evm.Create

refundGas()
applyTransactionFee()   (routes to coinbase or validator owner)
```

### `buyGas()` — TRC21 vs VIC

```go
func (st *StateTransition) buyGas() error {
    balanceTokenFee := st.msg.BalanceTokenFee()   // from Message, nil if not TRC21
    mgval := gasLimit × gasPrice

    if balanceTokenFee == nil {
        // Standard VIC path: deduct from sender now
        if senderBalance < mgval:
            return ErrInsufficientFunds
        statedb.SubBalance(sender, mgval)
    } else {
        // TRC21 path: check in-memory capacity, do NOT touch statedb
        if balanceTokenFee < mgval:
            return ErrInsufficientFunds
        // No SubBalance — fee is deferred to block-end UpdateTRC21Fee
    }
    gp.SubGas(gasLimit)
    st.gas = gasLimit
    st.initialGas = gasLimit
}
```

**Key:** TRC21 sponsorship means `SubBalance` is **never called** per transaction.
The in-memory `balanceFee` map pre-loaded from parent state is the only check;
statedb is not touched for gas at this point.

### `refundGas()`

```go
func (st *StateTransition) refundGas() {
    refund = min(gasUsed/2, statedb.GetRefund())
    st.gas += refund
    remaining = st.gas × gasPrice

    if balanceTokenFee == nil {
        // VIC tx: return unused gas to sender
        statedb.AddBalance(sender, remaining)
    }
    // TRC21 tx: NO refund to sender, NO statedb update
    // The remaining capacity is implicitly preserved in balanceFee (never deducted)

    gp.AddGas(st.gas)
}
```

**Key:** For TRC21 txs, `refundGas` makes no statedb change. The `balanceFee` in-memory
map was never decremented for gas upfront, so there is nothing to refund.

### `applyTransactionFee()`

```go
// After TIPTRC21Fee fork: fee routes to the validator's registered owner.
// Before the fork: fee goes to the block coinbase.
if block > TIPTRC21FeeBlock && ownerAddr != ZeroAddress:
    statedb.AddBalance(ownerAddr, gasUsed × gasPrice)
else:
    statedb.AddBalance(coinbase, gasUsed × gasPrice)
```

For TRC21 txs, `gasPrice` was overridden to `TRC21GasPrice` in `AsMessage`. The fee
added to the coinbase/owner corresponds to the TRC21 rate, not the user's original price.

---

## TRC21 Fee Accounting — End-to-End

### Where the fee capacity lives

```
Contract: TRC21IssuerSMC (0x8c0faeb5C6bEd2129b8674F262Fd45c4e9468bee)
  Storage layout:
    Slot 0: minCap         (uint256)
    Slot 1: tokens[]       (address[], dynamic array of registered tokens)
    Slot 2: tokensState    (mapping[tokenAddr] → feeCapacity)

Contract: TRC21Token (each token)
  Storage layout:
    Slot 0: balances       (mapping[addr] → uint256 token balance)
    Slot 1: minFee         (uint256 minimum fee per tx)
    Slot 2: issuer         (address of issuer/owner)
```

### `GetTRC21FeeCapacityFromState()` (called once per block from blockchain.go)

```
Read tokens[] length from TRC21IssuerSMC slot 1
For each token address in tokens[]:
    capacity = TRC21IssuerSMC.tokensState[tokenAddr]   (slot 2 mapping)
    if capacity > 0:
        result[tokenAddr] = capacity
return result   // map[tokenAddr → feeCapacity]
```

This map is the `balanceFee` parameter passed to `Process()`.

### Per-block lifecycle

```
Block start:
    balanceFee    = GetTRC21FeeCapacityFromStateWithCache(parent.Root, statedb)
    balanceUpdated = {}
    totalFeeUsed   = 0

Per tx (if TRC21):
    [no statedb change for gas]
    gasUsed, tokenFeeUsed = ApplyTransaction(...)
    fee = gasUsed × TRC21GasPrice
    balanceFee[token]    -= fee      // in-memory only
    balanceUpdated[token] = balanceFee[token]
    totalFeeUsed         += fee

Block end:
    UpdateTRC21Fee(statedb, balanceUpdated, totalFeeUsed):
        for token, newCap in balanceUpdated:
            TRC21IssuerSMC.tokensState[token] = newCap   // write capacity to storage
        TRC21IssuerSMC.nativeBalance -= totalFeeUsed     // deduct total gas paid
```

The net effect at block end:
- `TRC21IssuerSMC` storage reflects depleted capacity
- `TRC21IssuerSMC` native ETH/VIC balance decreases by `totalFeeUsed`
- Coinbase / validator owner gained that ETH/VIC through per-tx `applyTransactionFee`

---

## Transaction Type Reference

### Type 1: Standard VIC Transaction

A regular transaction. Not to a TRC21 contract.

```
buyGas():    SubBalance(sender, gasLimit × gasPrice)
EVM:         execute
refundGas(): AddBalance(sender, unused × gasPrice)
fee():       AddBalance(coinbase/owner, gasUsed × gasPrice)
```

**ETH net:** sender: `-gasUsed × gasPrice` | coinbase: `+gasUsed × gasPrice`

---

### Type 2: TRC21-Sponsored Transaction (pre-Atlas)

Transaction to a registered TRC21 token contract with sufficient fee capacity.

```
buyGas():    [no SubBalance — fee deferred to block end]
EVM:         execute
  SUCCESS → token transfer executes; issuer fee charged via EVM
  FAILURE → PayFeeWithTRC21TxFail(sender, token): charges min(balance, minFee) tokens
refundGas(): [no AddBalance — nothing was deducted]
fee():       AddBalance(coinbase/owner, gasUsed × TRC21GasPrice)
```

**ETH net per tx:**
- Sender: `0` ETH
- Coinbase/owner: `+gasUsed × TRC21GasPrice`

**Token net:**
- Success: user pays token fee via EVM execution (via transfer/transferFrom)
- Failure: `min(senderTokenBalance, minFee)` deducted from user, credited to issuer

**ETH net at block end (UpdateTRC21Fee):**
- `TRC21IssuerSMC.nativeBalance`: `-totalFeeUsed`
- This balances the per-tx coinbase gains: issuer's pre-deposit funds the gas

---

### Type 3: BlockSigner (0x89 — `ApplySignTransaction`)

Validator vote recording. Only before `TIPSigningBlock` (3,000,000). The BlockSigners
contract is deleted from state exactly at that block.

```
Verify sender using block signer
Nonce check + increment
Build receipt (GasUsed = 0)
Emit log at BlockSigners address
```

**No gas, no ETH movement, nonce incremented.**

---

### Type 4: Trading State Root (0x88 — `ApplyEmptyTransaction`)

Records the TradingStateDB Merkle root in block data. Active after `TIPTomoX` (20,581,700).

```
Build receipt (GasUsed = 0, Status = success)
Emit log at TradingStateAddr
```

**Data carrier only. No state mutation.**

The root is recovered during block validation by scanning block transactions for the
one sent to `TradingStateAddr` by the block author using `HomesteadSigner`.

---

### Type 5: TomoX Match Batch — `IsTradingTransaction()`

Order matching batch. Produced by the block author after running `ProcessOrderPending`.

```
ApplyEmptyTransaction():
    Build receipt (GasUsed = 0)
    Emit log

[Block validation re-runs CommitOrder for each order in the batch to verify state]
```

**No gas. Mutates both EVM statedb (token balances) and TradingStateDB (order books).**

---

### Type 6: Lending Apply / Finalized Trade

Same pattern as Type 5. Active after `TIPTomoXLending` (21,430,200).

```
ApplyEmptyTransaction() for both lending batch and finalized trade
```

---

## TomoX Order Processing

### Block production (miner path)

```
ProcessOrderPending(header, coinbase, chain, pendingOrders, statedb, tradingStateDB):
    Sort pending orders by nonce per address
    For each order:
        CommitOrder(header, coinbase, chain, statedb, tradingStateDB, orderBook, order)
            tomoxSnap = tradingStateDB.Snapshot()
            dbSnap    = statedb.Snapshot()
            trades, rejects, err = ApplyOrder(...)
            if err: revert both snapshots, return err
        accumulate trades, rejects
    return TxDataMatch[], MatchingResult{}
```

### Block validation (sync path)

During `ApplyTransaction` for a `IsTradingTransaction` tx:

```
DecodeTxMatchesBatch(tx.Data):
    HARD error if decode fails

For each order in batch:
    DecodeOrder(txDataMatch):       soft skip on failure
    CommitOrder(header, coinbase, chain, statedb, tradingStateDB, orderBook, order):
        HARD error if CommitOrder returns error
```

### `ApplyOrder()` — nonce and routing

```
nonce = tradingStateDB.GetNonce(order.UserAddress)
if nonce < order.Nonce: return ErrNonceTooHigh
if nonce > order.Nonce: return ErrNonceTooLow

tradingStateDB.SetNonce(order.UserAddress, nonce+1)   // nonce set BEFORE inner snapshot
tomoxSnap = tradingStateDB.Snapshot()                  // snapshot AFTER nonce increment
dbSnap    = statedb.Snapshot()

if VerifyOrder(statedb) fails:
    rejects += order
    return nil, rejects, nil   // soft reject; nonce NOT reverted (path A)

if order.Status == Cancelled:
    ProcessCancelOrder(...)
    return

route to matching engine (market or limit)
```

**Nonce behavior (Path A):** Early exits from soft-rejects (VerifyOrder fail, invalid
price/quantity) do NOT revert the nonce increment. The nonce remains at `nonce+1`.
This matches victionchain behavior — it is intentional.

---

## Key Addresses and Constants

```
TRC21IssuerSMC:      0x8c0faeb5C6bEd2129b8674F262Fd45c4e9468bee
BlockSigners:        0x0000000000000000000000000000000000000089  (deleted at block 3,000,000)
TradingStateAddr:    0x0000000000000000000000000000000000000088
TomoXAddr (0x91):    (internal matching contract address)

TIPSigning:          3,000,000
TIPRandomize:        3,464,000
BlackListHFBlock:    9,349,100
TIPTRC21Fee:         13,523,400
TIPTomoX:            20,581,700
TIPTomoXLending:     21,430,200
Atlas:               97,705,094
```

---

## How This Differs from vic-geth

| Aspect | victionchain (legacy) | vic-geth |
|---|---|---|
| TRC21 fee capacity | Pre-loaded once per block into `balanceFee` map before `Process()` | Read from statedb per transaction in `vrc25BuyGas()` |
| Gas buy for sponsored tx | Skip `SubBalance` entirely; in-memory map is the guard | `SubBalance(VRC25Contract, gasLimit × VRC25GasPrice)` per tx |
| Gas refund for sponsored tx | No-op; nothing was deducted | `AddBalance(VRC25Contract, unused × VRC25GasPrice)` per tx |
| Fee capacity write-back | Batch at block end via `UpdateTRC21Fee` (sets storage + SubBalance) | Per-tx via `vrc25BuyGas/RefundGas` (no block-end batch needed) |
| `gasPrice` for sponsored tx | Overridden in `AsMessage()` to `TRC21GasPrice` | Overridden in `vrc25BuyGas()` via `st.gasPrice = VRC25GasPrice` |
| Failed-tx penalty | `PayFeeWithTRC21TxFail()` after `ApplyMessage`, inside `ApplyTransaction` | `PayFeeWithVRC25()` in `afterApplyTransaction` hook |
| TradingStateDB passing | Explicit parameter through `Process()` → `ApplyTransaction()` → EVM | Held on `victionProcessorState.tradingStateDB`, injected per block in `beforeProcess` |
| System tx routing | Inside `ApplyTransaction()` before EVM | In `applyVictionTransaction()` hook, before `applyTransaction()` |
| Fee routing destination | `ApplyMessage(... coinbaseOwner)` → `applyTransactionFee` inside `TransitionDb` | `applyTransactionFee` in `state_transition_viction.go`, called from standard `TransitionDb` |
