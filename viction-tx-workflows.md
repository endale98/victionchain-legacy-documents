# Viction Transaction Workflows

Covers the full per-transaction gas and fee lifecycle for both **vic-geth** (current)
and **victionchain** (legacy reference). Use this when reasoning about sponsorship
correctness, fee routing, or porting logic between the two clients.

---

## Hardfork Reference

| Hardfork | Block | Effect |
|---|---|---|
| TIPSigning | 3,000,000 | BlockSigner contract deleted from state |
| BlackListHF | 9,349,100 | Blacklist validation enabled |
| TIPTRC21Fee | 13,523,400 | Fee routed to validator owner instead of coinbase |
| TIPTomoX | 20,581,700 | TomoX order transactions enabled |
| Atlas | 97,705,094 | VRC25GasPrice introduced; VRC25 storage layout upgraded |

---

## vic-geth

### Block-level hooks (`state_processor_viction.go`)

```
beforeProcess(block)
  - Delete BlockSigner contract at TIPSigning block
  - Apply VRC25 storage upgrade at Atlas block
  - Apply Saigon hardfork state changes
  - Pre-cache tx signers in parallel

for each tx:
  beforeApplyTransaction(tx)     ← blacklist check, bypass balances for legacy blocks
  applyVictionTransaction(tx)    ← handle system txs (BlockSigner 0x89); returns (handled, receipt)
  if not handled:
    ApplyTransaction(tx)         ← standard EVM path (see per-tx flow below)
  afterApplyTransaction(tx)      ← failed-tx VRC25 token penalty (pre-Atlas only)

afterProcess(block)              ← currently no-op
```

### Per-transaction flow (`state_transition.go` + `state_transition_viction.go`)

```
preCheck()
  └─ buyGas()
       └─ vrc25BuyGas()      ← sets st.payer and st.gasPrice if sponsored

EVM executes (Call / Create)

refundGas()
  isCustomGasRefunding = isVRC25Transaction() || IsAtlas
  ├─ true  → vrc25RefundGas(remaining)
  └─ false → AddBalance(msg.From, remaining)

applyTransactionFee()
```

### `vrc25BuyGas()` logic

```
st.payer = msg.From   (default)

feeCap = storage[VRC25Contract][tokensState[msg.To]]

if feeCap == nil:          → contract creation, no sponsorship — return
if feeCap < gasLimit×P:    → insufficient capacity, fallback to sender — return

storage[tokensState[msg.To]] -= gasLimit × P
st.gasPrice = P                   ← P = VRC25GasPrice
st.payer    = VRC25Contract
```

After `vrc25BuyGas`, `buyGas()` always does:
```
SubBalance(st.payer, gasLimit × st.gasPrice)
```
So for sponsored txs: `SubBalance(VRC25Contract, gasLimit × P)`.

### `vrc25RefundGas(remaining)` logic

```
if isVRC25Transaction():
  storage[tokensState[msg.To]] += remaining   ← restore unused capacity

AddBalance(st.payer, remaining)               ← return native ETH to whoever paid
```

`isVRC25Transaction()` = `st.payer != msg.From`.
For Atlas non-sponsored txs the storage write is skipped; only the ETH refund runs.

### `applyTransactionFee()` logic

```
txFee = gasUsed × st.gasPrice

if victionCfg == nil:
  AddBalance(coinbase, txFee)   ← non-Viction chain
  return

if IsAtlas && isVRC25Transaction() && VRC25GasPrice != nil:
  txFee = gasUsed × VRC25GasPrice   ← defensive recalculate (already correct)

if not IsTIPTRC21Fee:
  AddBalance(coinbase, txFee)
  return

owner = state[ValidatorContract][slot(coinbase)]
if owner != zero:
  AddBalance(owner, txFee)
```

### `afterApplyTransaction()` logic

```
if not IsAtlas && tx.To != nil && IsTIPTRC21Fee && receipt.Status == Failed:
  feeCap = storage[VRC25Contract][tokensState[tx.To]]
  if feeCap > 0:
    PayFeeWithVRC25(msg.From, tx.To)
    → deducts minFee from user's token balance, credits token issuer
```

Only fires pre-Atlas. Post-Atlas this path is disabled.

---

### Scenario Flows

#### 1. Pre-Atlas, non-sponsored

```
vrc25BuyGas:  feeCap == 0 → payer = msg.From, gasPrice = tx.gasPrice
buyGas:       SubBalance(msg.From, gasLimit × gasPrice)
EVM:          executes
refundGas:    isCustomGasRefunding = false
              AddBalance(msg.From, remaining)
applyFee:     IsTIPTRC21Fee? AddBalance(validatorOwner, gasUsed × gasPrice)
                          : AddBalance(coinbase, gasUsed × gasPrice)
afterApply:   nothing
```

#### 2. Pre-Atlas, VRC25-sponsored, success

```
vrc25BuyGas:  storage[tokensState[contract]] -= gasLimit × P
              payer = VRC25Contract, gasPrice = P
buyGas:       SubBalance(VRC25Contract, gasLimit × P)
EVM:          executes (success)
refundGas:    isCustomGasRefunding = true
              storage[tokensState[contract]] += remaining
              AddBalance(VRC25Contract, remaining)
applyFee:     AddBalance(validatorOwner, gasUsed × P)
afterApply:   receipt.Status == Success → nothing

Net: VRC25Contract −(gasUsed × P) in both storage and native ETH.
```

#### 3. Pre-Atlas, VRC25-sponsored, failed tx

```
vrc25BuyGas:  same deductions as success case
buyGas:       SubBalance(VRC25Contract, gasLimit × P)
EVM:          reverts (token transfer never executes)
refundGas:    storage += remaining, AddBalance(VRC25Contract, remaining)
applyFee:     AddBalance(validatorOwner, gasUsed × P)
afterApply:   receipt.Status == Failed, IsTIPTRC21Fee, feeCap > 0
              PayFeeWithVRC25(msg.From, contract)
              → user's token balance −minFee, issuer's token balance +minFee

Net: VRC25Contract pays gas in ETH. User pays a token penalty.
```

#### 4. Post-Atlas, non-sponsored

```
vrc25BuyGas:  feeCap == 0 → payer = msg.From, gasPrice = tx.gasPrice
buyGas:       SubBalance(msg.From, gasLimit × gasPrice)
EVM:          executes
refundGas:    isCustomGasRefunding = true (IsAtlas)
              isVRC25Transaction == false → skip storage
              AddBalance(msg.From, remaining)       ← identical to non-Atlas path
applyFee:     AddBalance(validatorOwner, gasUsed × gasPrice)
afterApply:   IsAtlas → condition false → nothing
```

#### 5. Post-Atlas, VRC25-sponsored

```
vrc25BuyGas:  storage[tokensState[contract]] -= gasLimit × P
              payer = VRC25Contract, gasPrice = P
buyGas:       SubBalance(VRC25Contract, gasLimit × P)
EVM:          executes
refundGas:    isCustomGasRefunding = true
              isVRC25Transaction == true → storage += remaining
              AddBalance(VRC25Contract, remaining)
applyFee:     txFee recalculated with VRC25GasPrice (defensive; already correct)
              AddBalance(validatorOwner, gasUsed × P)
afterApply:   IsAtlas → condition false → nothing (no token penalty post-Atlas)
```

### ETH accounting summary

| Scenario | Who pays | Net ETH cost | Token cost |
|---|---|---|---|
| Non-sponsored | msg.From | gasUsed × tx.gasPrice | none |
| Sponsored, success | VRC25Contract | gasUsed × VRC25GasPrice | none |
| Sponsored, failed (pre-Atlas) | VRC25Contract | gasUsed × VRC25GasPrice | user pays minFee in token |
| Sponsored, failed (post-Atlas) | VRC25Contract | gasUsed × VRC25GasPrice | none |

---

## victionchain (legacy)

### Block-level (`state_processor.go`)

```
balanceFee = pre-loaded map[tokenAddr → feeCapacity] from state   ← pre-Atlas only
             (post-Atlas: read fresh from state per-tx instead)

parentState = statedb.Copy()   ← snapshot for potential revert

for each tx:
  blacklist check
  TomoZ/TomoX apply-tx validation
  ApplyTransaction(config, balanceFee, ...)
  if tokenFeeUsed && !isAtlas:
    balanceFee[tx.To] -= gasUsed × TRC21GasPrice   ← update in-memory map
    balanceUpdated[tx.To] = balanceFee[tx.To]
    totalFeeUsed += gasUsed × TRC21GasPrice

if !isAtlas:
  UpdateTRC21Fee(statedb, balanceUpdated, totalFeeUsed)
  → batch-writes balanceUpdated to storage slots
  → SubBalance(TRC21IssuerSMC, totalFeeUsed)
```

The `balanceFee` map is the block-scope accounting mechanism for pre-Atlas sponsored fees.
Post-Atlas, fee capacity is written directly per-tx (no batch at end of block).

### Per-transaction flow

```
preCheck(useAtlasRule)
  └─ buyGas(useAtlasRule)
       ├─ checkBalance(balanceTokenFee, useAtlasRule)
       └─ subtractBalance(balanceTokenFee, useAtlasRule) → isUsedTokenFee bool

EVM executes

refundGas(isUsedTokenFee)

transactionFee = gasUsed × gasPrice
if isAtlas && isUsedTokenFee:
  transactionFee = gasUsed × TRC21GasPrice

if block > TIPTRC21FeeBlock:
  AddBalance(coinbaseOwner, transactionFee)
else:
  AddBalance(coinbase, transactionFee)
```

### `buyGas` / `subtractBalance` logic

**Pre-Atlas:**
```
balanceTokenFee = balanceFee[tx.To]   ← from pre-loaded map

if balanceTokenFee == nil:
  SubBalance(sender, gasLimit × gasPrice)   ← regular tx
  isUsedTokenFee = false
else if balanceTokenFee < gasLimit × gasPrice:
  error: insufficient balance            ← pre-Atlas requires full VIC equivalent
else:
  // sponsored — do NOT SubBalance anyone
  isUsedTokenFee = true
```

Pre-Atlas sponsored txs: **no ETH is deducted at buyGas time**. The fee capacity is
tracked only in the in-memory `balanceFee` map, flushed to state at end of block.

**Post-Atlas:**
```
balanceTokenFee = GetTRC21FeeCapacityFromStateWithToken(statedb, tx.To)   ← live from state

vrc25val = gasLimit × TRC21GasPrice

if balanceTokenFee == nil || balanceTokenFee <= vrc25val:
  SubBalance(sender, gasLimit × gasPrice)
  isUsedTokenFee = false
else:
  vrc25PayGas(token, vrc25val):
    storage[tokensState[token]] -= vrc25val
    SubBalance(TRC21IssuerSMC, vrc25val)
  isUsedTokenFee = true
```

Post-Atlas sponsored txs: ETH deducted from `TRC21IssuerSMC` immediately at buyGas.

### `refundGas(isUsedTokenFee)` logic

**Pre-Atlas:**
```
if balanceTokenFee == nil:
  AddBalance(sender, remaining)     ← regular tx refund
// sponsored: no refund (no ETH was charged)
```

**Post-Atlas:**
```
if isUsedTokenFee:
  vrc25RefundGas(token, remaining):
    storage[tokensState[token]] += remaining
    AddBalance(TRC21IssuerSMC, remaining)
// non-sponsored post-Atlas: no refund
```

> Note: post-Atlas non-sponsored txs in victionchain receive no gas refund.
> The sender pays for the full `gasLimit × gasPrice` regardless of actual usage.

### Failed-tx token penalty

In `ApplyTransaction`, after EVM execution:
```
fee = gasUsed × gasPrice
if block > TIPTRC21FeeBlock:
  fee = gasUsed × TRC21GasPrice

if balanceFee != nil && balanceFee > fee && failed:
  PayFeeWithTRC21TxFail(statedb, msg.From, tx.To)
  → deducts minFee from user's token balance
```

This fires for both pre-Atlas and post-Atlas sponsored failed txs.

---

## Side-by-Side Comparison

| | vic-geth | victionchain |
|---|---|---|
| **Sponsorship detection** | `GetFeeCapacity` from storage per-tx | pre-Atlas: pre-loaded `balanceFee` map; post-Atlas: `GetTRC21FeeCapacityFromStateWithToken` per-tx |
| **Pre-Atlas buy gas (sponsored)** | deducts storage + `SubBalance(VRC25Contract)` | no ETH deducted; in-memory map only |
| **Pre-Atlas refund (sponsored)** | restores storage + `AddBalance(VRC25Contract)` | no refund (no ETH was charged) |
| **Pre-Atlas fee flush** | none needed (per-tx accounting) | `UpdateTRC21Fee` batch-writes all updates at end of block |
| **Post-Atlas buy gas (sponsored)** | deducts storage + `SubBalance(VRC25Contract)` | deducts storage + `SubBalance(TRC21IssuerSMC)` |
| **Post-Atlas refund (sponsored)** | restores storage + `AddBalance(VRC25Contract)` | restores storage + `AddBalance(TRC21IssuerSMC)` |
| **Post-Atlas refund (non-sponsored)** | `AddBalance(msg.From, remaining)` — standard | no refund |
| **Failed-tx token penalty** | pre-Atlas only (`afterApplyTransaction`) | both pre- and post-Atlas (`ApplyTransaction`) |
| **Fee recipient** | `ValidatorContract` owner after TIPTRC21Fee; coinbase before | `coinbaseOwner` (from `statedb.GetOwner`) after TIPTRC21Fee; coinbase before |
| **Gas price override** | `st.gasPrice = VRC25GasPrice` in `vrc25BuyGas` | `transactionFee` recalculated after EVM; gasPrice not overridden |
| **System contract address** | `VRC25Contract` (from `VictionConfig`) | `TRC21IssuerSMC` (hardcoded `common` constant) |
| **Atlas non-sponsored refund path** | `vrc25RefundGas` (storage write skipped) | no refund |

### Key architectural difference

victionchain uses a **block-scope batch model** pre-Atlas: sponsored fees accumulate in
an in-memory map and are flushed in one `UpdateTRC21Fee` call at the end of the block.
This means mid-block queries to storage see stale fee capacities.

vic-geth uses a **per-transaction model** for all forks: each sponsored tx updates the
storage slot and native ETH balance atomically within its own `vrc25BuyGas` /
`vrc25RefundGas` calls. Mid-block fee capacity is always accurate.
