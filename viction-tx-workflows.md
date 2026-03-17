# Viction Transaction Workflows

This document traces the complete execution path for every transaction type in `vic-geth`,
from block entry through state finalization. Covers hook dispatch, gas accounting, fee routing,
and the VRC25 sponsorship mechanism.

---

## Block Processing Loop

`core/state_processor.go: Process()`

```
for each tx in block:
    1. msg, err = tx.AsMessage(signer)
    2. beforeApplyTransaction(block, tx, msg, statedb)       ← viction hook
    3. statedb.Prepare(tx.Hash, block.Hash, i)
    4. handled, receipt, _, err, _ = applyVictionTransaction(...)  ← viction hook
    5. if !handled:
           receipt, err = applyTransaction(msg, ...)          ← standard EVM path
    6. afterApplyTransaction(tx, msg, statedb, receipt, ...)  ← viction hook

p.engine.Finalize(...)
afterProcess(block, statedb)                                  ← viction hook
```

---

## Hook Reference

All hooks live in `core/state_processor_viction.go`.

| Hook | When | Purpose |
|---|---|---|
| `beforeProcess` | Start of block | Initialize `victionState`, apply hardfork state mutations, open TradingStateDB |
| `beforeApplyTransaction` | Before each tx | Blacklist check, legacy balance bypass |
| `applyVictionTransaction` | Before EVM | Route system transactions (0x89 / 0x91–0x94) |
| `afterApplyTransaction` | After each tx | VRC25 failed-tx fee charge |
| `afterProcess` | End of block | TomoX trading root verification |

---

## Transaction Types

### 1. Normal EVM Transaction

A regular user transaction calling a non-system contract (or ETH transfer).

**`applyVictionTransaction` returns:** `handled = false`

**Full path: `applyTransaction` → `TransitionDb()`**

```
preCheck()
  └─ vrc25BuyGas()
       st.payer = msg.From()          // default: sender pays
       feeCap = GetFeeCapacity(...)   // check if contract is VRC25-sponsored
       if feeCap < vrc25GasFee:       // insufficient or not sponsored
           return nil                 // payer stays as sender, gasPrice unchanged
  └─ buyGas()
       mgval = gasLimit × msg.GasPrice()
       SubBalance(sender, mgval)      // sender pays upfront
       st.gas = gasLimit

IntrinsicGas check

EVM execution (Call or Create)
  → st.gas decremented by opcodes

refundGas()
  refund = min(gasUsed/2, stateRefund)
  st.gas += refund
  remaining = st.gas × gasPrice
  isCustomGasRefunding = false (pre-Atlas non-sponsored)
  → AddBalance(sender, remaining)    // refund unused gas to sender
  gp.AddGas(st.gas)                  // return to block gas counter

applyTransactionFee()
  txFee = gasUsed() × gasPrice
  if IsTIPTRC21Fee: AddBalance(validatorOwner, txFee)
  else:             AddBalance(coinbase, txFee)
```

**Net ETH balance changes:**
- Sender: `-gasUsed × gasPrice`
- Coinbase / validator owner: `+gasUsed × gasPrice`

---

### 2. VRC25-Sponsored Transaction (pre-Atlas)

A transaction to a VRC25 token contract where the contract has enough pre-deposited
fee capacity to cover the gas cost. Gas is "free" for the sender in ETH terms; the
VRC25 contract absorbs the gas cost in native ETH from its pre-deposit.

**`applyVictionTransaction` returns:** `handled = false` (normal EVM path)

```
preCheck()
  └─ vrc25BuyGas()
       st.payer = msg.From()            // set default first
       feeCap = GetFeeCapacity(statedb, VRC25Contract, tx.To())
       vrc25GasFee = gasLimit × VRC25GasPrice
       if feeCap >= vrc25GasFee:        // contract can cover gas
           storage[tx.To()] -= vrc25GasFee   // deduct from fee capacity slot
           st.gasPrice = VRC25GasPrice
           st.payer   = VRC25Contract
  └─ buyGas()
       mgval = gasLimit × VRC25GasPrice
       SubBalance(VRC25Contract, mgval)  // sponsor pays upfront
       st.gas = gasLimit

IntrinsicGas check

EVM execution
  → on SUCCESS: token transfer executes, user pays token fee to issuer via EVM
  → on FAILURE: EVM reverts, no token transfer, no token fee paid

refundGas()
  refund = min(gasUsed/2, stateRefund)
  st.gas += refund
  remaining = st.gas × VRC25GasPrice
  isCustomGasRefunding = true (isVRC25Transaction())
  → vrc25RefundGas(remaining):
       storage[tx.To()] += remaining          // restore unused capacity to slot
       AddBalance(VRC25Contract, remaining)   // restore unused native ETH
  gp.AddGas(st.gas)

applyTransactionFee()
  txFee = gasUsed() × VRC25GasPrice
  → AddBalance(validatorOwner or coinbase, txFee)

afterApplyTransaction()  [state_processor_viction.go]
  if !IsAtlas && IsTIPTRC21Fee && receipt.Status == FAILED:
      feeCap = GetFeeCapacity(statedb, VRC25Contract, tx.To())
      if feeCap > 0:
          PayFeeWithVRC25(statedb, sender, tx.To())
          // charges min(senderTokenBalance, minFee) tokens
          // deducts from sender's token balance, credits issuer
          // reason: tx failed so EVM never executed the token fee
```

**Net ETH balance changes:**
- Sender: `0` (no ETH spent)
- VRC25Contract: `-actualGasUsed × VRC25GasPrice`
- Coinbase / validator owner: `+actualGasUsed × VRC25GasPrice`

**Net VRC25 storage change:**
- `feeCap[tx.To()]`: `-actualGasUsed × VRC25GasPrice`

**Token balance change (success):** user pays fee via EVM execution of the transfer
**Token balance change (failure):** `PayFeeWithVRC25` deducts `min(balance, minFee)` from sender

---

### 3. VRC25-Sponsored Transaction (post-Atlas)

Identical to the pre-Atlas path through `TransitionDb()`. The differences are:

- `ApplyVIPVRC25Upgrade` runs in `beforeProcess` at the Atlas block, migrating VRC25
  contract state.
- `applyTransactionFee()` uses `VRC25GasPrice` for VRC25 txs (explicit recalculation
  at line 93 of `state_transition_viction.go`).
- `afterApplyTransaction` fee tracking is gated by `!IsAtlas` — no block-level fee
  accumulation at Atlas+, since storage and ETH are handled per-tx.

---

### 4. Non-Sponsored Atlas Transaction

A regular transaction at an Atlas-height block where `IsAtlas == true` but the
transaction is NOT VRC25-sponsored (either not a VRC25 contract, or fee capacity
was insufficient).

The only Atlas-specific difference is in `refundGas()`:

```
isCustomGasRefunding = false || IsAtlas = true
→ vrc25RefundGas(remaining):
     isVRC25Transaction() == false → skip storage update
     AddBalance(sender, remaining)  // normal ETH refund
```

**Net:** identical to a normal EVM transaction (Section 1).

---

### 5. BlockSigner Transaction (0x89 — `applySignTransaction`)

Validator signature recording. Active only after `TIPSigningBlock`. Deleted from state
at `TIPSigningBlock` via `beforeProcess`.

**`applyVictionTransaction` returns:** `handled = true, receipt (GasUsed=0)`

```
applySignTransaction():
    Finalise or IntermediateRoot (Byzantium check)
    Verify sender from signer
    Nonce check (ErrNonceTooHigh / ErrNonceTooLow)
    statedb.SetNonce(sender, nonce+1)
    Build receipt: GasUsed=0, Status=success
    Emit log at ValidatorBlockSignContract
```

**No gas deducted. No ETH movement. Nonce incremented.**

---

### 6. TomoX Order Match (0x91 — `applyTomoXTx`)

Batch of order executions. Active pre-Atlas only, after `TIPTomoX`.

**`applyVictionTransaction` returns:** `handled = true, receipt (GasUsed=0)`

```
applyTomoXTx():
    Finalise or IntermediateRoot (Byzantium check)
    if tx.Data != nil && tradingStateDB != nil && tradingEngine != nil:
        txMatchBatch = DecodeTxMatchesBatch(tx.Data)  // HARD error on failure
        for each match in batch:
            order = txDataMatch.DecodeOrder()          // soft skip on failure (log.Warn)
            orderBook = GetTradingOrderBookHash(base, quote)
            trades, rejects, err = CommitOrder(header, coinbase, bc,
                                               statedb, tradingStateDB,
                                               orderBook, order)        // HARD error
    Build receipt: GasUsed=0
    Emit log at TomoXContract
```

**No gas deducted. Mutates both EVM statedb (token balances) and TradingStateDB (order books).**

---

### 7. Trading State Root Commit (0x92 — `applyEmptyTransaction`)

Encodes the TradingStateDB Merkle root into the block. Active pre-Atlas only,
after `TIPTomoX`. Verified at block end by `afterProcess` via `GetTradingStateRoot`.

**`applyVictionTransaction` returns:** `handled = true, receipt (GasUsed=0)`

```
applyEmptyTransaction():
    Finalise or IntermediateRoot (Byzantium check)
    Build receipt: GasUsed=0
    Emit log at TradingStateContract
```

**No state mutation. Acts as a data carrier for the trading root.**

`GetTradingStateRoot()` recovery logic (called from `afterProcess`):

```
for each tx in block:
    if tx.To() == TradingStateContract:
        from = Sender(HomesteadSigner, tx)  // 0x92 always uses HomesteadSigner
        if from == blockAuthor:
            return tx.Data()[:32]           // first 32 bytes = trading root
return EmptyRoot
```

---

### 8. Lending Batch / Finalized (0x93 / 0x94 — `applyEmptyTransaction`)

Same as 0x92 but for the lending engine. Active pre-Atlas only, after `TIPTomoXLending`.
No state mutation. Data carrier only.

---

## VRC25 Fee Capacity Accounting (Summary)

```
Per-transaction (state_transition_viction.go):
  vrc25BuyGas():
    storage[tx.To() in VRC25Contract] -= gasLimit × VRC25GasPrice
    payer = VRC25Contract
    gasPrice = VRC25GasPrice
  buyGas() in state_transition.go:
    SubBalance(VRC25Contract, gasLimit × VRC25GasPrice)
  vrc25RefundGas():
    storage[tx.To() in VRC25Contract] += unused × VRC25GasPrice   ← iff sponsored
    AddBalance(payer, unused × VRC25GasPrice)

Net after transaction:
  storage[token]: -actualGasUsed × VRC25GasPrice
  VRC25Contract ETH: -actualGasUsed × VRC25GasPrice
  validator/coinbase ETH: +actualGasUsed × VRC25GasPrice

No block-end batch update (UpdateFeeCapacity) is needed or called.
The per-tx mechanism is self-contained.
```

---

## afterProcess — TomoX Root Verification

Runs after all transactions for pre-Atlas blocks with an active trading engine.

```
afterProcess():
    gotRoot  = tradingStateDB.IntermediateRoot()
    author   = engine.Author(block.Header)
    expected = GetTradingStateRoot(block, TradingStateContract, author)
    if gotRoot != expected:
        return HARD ERROR  // consensus-critical: block is rejected
    log.Debug("trading state root verified")
```

---

## Error Handling Conventions

| Location | Error type | Reason |
|---|---|---|
| `beforeProcess` — `GetTradingState` fails | Hard error | nil tradingStateDB skips root check → invalid blocks accepted |
| `applyTomoXTx` — batch decode fails | Hard error | Cannot replay trading state deterministically |
| `applyTomoXTx` — per-order decode fails | Soft skip (`log.Warn`) | Matches victionchain; individual bad orders don't abort the block |
| `applyTomoXTx` — `CommitOrder` fails | Hard error | Matches victionchain; order engine failure means state is corrupt |
| `afterProcess` — root mismatch | Hard error | Consensus-critical: mismatched roots invalidate the block |
| `applySignTransaction` — sender error | Returns error in receipt | Treated as invalid tx, block is rejected |
