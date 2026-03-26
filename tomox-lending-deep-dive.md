# TomoX Lending – Deep Dive (Legacy VictionChain)

> **Source codebase**: `victionchain/tomoxlending/`, `victionchain/core/types/`, `victionchain/contracts/tomox/`
>
> **Written**: 2026-03-26 — based on code-level analysis of the legacy TomoChain implementation.

---

## Table of Contents

1. [Overview & Architecture](#1-overview--architecture)
2. [Core Data Structures](#2-core-data-structures)
3. [Smart Contract Layer (LendingRegistration)](#3-smart-contract-layer-lendingregistration)
4. [Order Lifecycle](#4-order-lifecycle)
5. [Matching Engine](#5-matching-engine)
6. [Settlement & Balance Management](#6-settlement--balance-management)
7. [Interest Rate Calculation](#7-interest-rate-calculation)
8. [Collateral & Price Oracle](#8-collateral--price-oracle)
9. [Liquidation System](#9-liquidation-system)
10. [TopUp / Recall Mechanisms](#10-topup--recall-mechanisms)
11. [Repay Flow](#11-repay-flow)
12. [Cancel Flow](#12-cancel-flow)
13. [State Storage (Trie Architecture)](#13-state-storage-trie-architecture)
14. [SDK Node Sync & Reorg Handling](#14-sdk-node-sync--reorg-handling)
15. [Fee Structure](#15-fee-structure)
16. [Key Constants & Addresses](#16-key-constants--addresses)
17. [File Index](#17-file-index)

---

## 1. Overview & Architecture

TomoX Lending is a **decentralized peer-to-peer lending protocol** built as an extension of the TomoX DEX engine. It runs **inside the consensus layer** (not via smart contract calls from external RPC) — the matching engine executes during block production, directly modifying EVM state.

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Block Producer                       │
│  ┌──────────────┐   ┌──────────────────┐   ┌─────────────┐ │
│  │  Lending Pool │──▶│  Order Processor  │──▶│  State DB   │ │
│  │  (pending txs)│   │  (matching engine)│   │  (EVM state)│ │
│  └──────────────┘   └──────────────────┘   └─────────────┘ │
│                            │                      │         │
│                     ┌──────┴──────┐        ┌──────┴───────┐ │
│                     │ LendingState │        │ TradingState │ │
│                     │   (trie DB)  │        │   (trie DB)  │ │
│                     └─────────────┘        └──────────────┘ │
└─────────────────────────────────────────────────────────────┘
        │                                            │
        ▼                                            ▼
  ┌──────────────┐                         ┌─────────────────┐
  │  MongoDB/SDK  │                         │ LendingRegistry │
  │  (off-chain)  │                         │ (Smart Contract)│
  └──────────────┘                         └─────────────────┘
```

### Two Participants

| Role       | Action                                             |
|------------|----------------------------------------------------|
| **Investor** (Lender)  | Deposits lending tokens to earn interest   |
| **Borrower**           | Locks collateral to receive lending tokens |

### Key Concepts

- **Lending Token**: The token being lent (e.g., USDT, TOMO).
- **Collateral Token**: The token the borrower locks (e.g., BTC, ETH).
- **Term**: Loan duration in seconds (e.g., 86400 = 1 day, 604800 = 7 days).
- **Interest Rate (APR)**: Annual Percentage Rate, stored as integer (e.g., `10` = 10%).
- **Deposit Rate**: Over-collateralization ratio (e.g., `150` = 150%).
- **Liquidation Rate**: Threshold where collateral is liquidated (e.g., `110` = 110%).
- **Recall Rate**: Threshold for partial collateral recall (e.g., `200` = 200%).

---

## 2. Core Data Structures

### LendingItem (Order)
**File**: `tomoxlending/lendingstate/lendingitem.go`

```go
type LendingItem struct {
    Quantity        *big.Int       // Amount of lending token
    Interest        *big.Int       // Interest rate (APR)
    Side            string         // "INVEST" or "BORROW"
    Type            string         // "LO" (Limit), "MO" (Market), "REPAY", "TOPUP", "RECALL"
    LendingToken    common.Address // Token being lent
    CollateralToken common.Address // Token used as collateral (borrower only)
    AutoTopUp       bool           // Enable auto top-up on liquidation threshold
    FilledAmount    *big.Int       // How much has been matched
    Status          string         // NEW, OPEN, FILLED, PARTIAL_FILLED, CANCELLED, REJECTED
    Relayer         common.Address // Relayer coinbase processing this order
    Term            uint64         // Loan duration in seconds
    UserAddress     common.Address // User who placed the order
    Signature       *Signature     // ECDSA signature (V, R, S)
    Hash            common.Hash    // Order hash
    LendingId       uint64         // Auto-increment ID within the order book
    LendingTradeId  uint64         // References a trade (for REPAY/TOPUP)
    Nonce           *big.Int       // User nonce
}
```

**Order statuses**:
- `NEW` → submitted, not yet processed
- `OPEN` → inserted into the order book (unmatched/partially matched)
- `PARTIAL_FILLED` → partially matched
- `FILLED` → fully matched
- `CANCELLED` → cancelled by user
- `REJECTED` → rejected (insufficient balance, invalid pair, etc.)

### LendingTrade (Matched Loan)
**File**: `tomoxlending/lendingstate/trade.go`

```go
type LendingTrade struct {
    Borrower               common.Address // Borrower address
    Investor               common.Address // Investor/lender address
    LendingToken           common.Address // Token being lent
    CollateralToken        common.Address // Token locked as collateral
    BorrowingOrderHash     common.Hash    // Hash of the borrowing order
    InvestingOrderHash     common.Hash    // Hash of the investing order
    BorrowingRelayer       common.Address // Borrower's relayer
    InvestingRelayer       common.Address // Investor's relayer
    Term                   uint64         // Loan duration in seconds
    Interest               uint64         // APR
    CollateralPrice        *big.Int       // Price at time of trade
    LiquidationPrice       *big.Int       // Price that triggers liquidation
    CollateralLockedAmount *big.Int       // Amount of collateral locked
    AutoTopUp              bool           // Whether auto top-up is enabled
    LiquidationTime        uint64         // Unix timestamp when loan expires
    DepositRate            *big.Int       // Deposit rate at time of trade
    LiquidationRate        *big.Int       // Liquidation rate at time of trade
    RecallRate             *big.Int       // Recall rate at time of trade
    Amount                 *big.Int       // Lent amount
    BorrowingFee           *big.Int       // Fee charged to borrower
    InvestingFee           *big.Int       // Fee charged to investor (currently 0)
    Status                 string         // OPEN, CLOSED, LIQUIDATED
    TradeId                uint64         // Auto-increment trade ID
    Hash                   common.Hash    // keccak256(InvestingOrderHash, BorrowingOrderHash)
}
```

**Trade statuses**:
- `OPEN` → active loan
- `CLOSED` → successfully repaid
- `LIQUIDATED` → liquidated (by price or time)

### LendingTransaction (Protocol-level Transaction)
**File**: `core/types/lending_transaction.go`

A separate transaction type in the mempool, NOT a regular Ethereum transaction. Contains all the fields needed for a lending operation, signed by the user's private key.

---

## 3. Smart Contract Layer (LendingRegistration)

**File**: `contracts/tomox/contract/LendingRegistration.sol`
**Address**: `common.LendingRegistrationSMC`

This Solidity contract governs protocol configuration. The Go code reads its state directly via `statedb.GetState()` (NOT via contract calls).

### Storage Layout

| Slot | Variable | Description |
|------|----------|-------------|
| 0 | `LENDINGRELAYER_LIST` | `mapping(address => LendingRelayer)` — relayer configs |
| 1 | `COLLATERAL_LIST` | `mapping(address => Collateral)` — collateral params |
| 2 | `COLLATERALS` | `address[]` — default collateral tokens |
| 3 | `BASES` | `address[]` — supported lending tokens |
| 4 | `TERMS` | `uint256[]` — supported loan terms |
| 5 | `ILO_COLLATERALS` | `address[]` — ILO collateral tokens (not used in v2.2) |

### LendingRelayer Struct
```solidity
struct LendingRelayer {
    uint16 _tradeFee;     // Borrow fee rate (0-999 = 0%-9.99%)
    address[] _baseTokens; // Supported lending tokens
    uint256[] _terms;      // Supported loan terms (in seconds)
    address[] _collaterals; // Per-pair collaterals (unused, falls back to default)
}
```

### Collateral Struct
```solidity
struct Collateral {
    uint256 _depositRate;     // e.g., 150 = 150% over-collateralization
    uint256 _liquidationRate; // e.g., 110 = 110% liquidation threshold
    uint256 _recallRate;      // e.g., 200 = 200% recall threshold
    mapping(address => Price) _price; // Price per lending token pair
}
```

### Key Functions

| Function | Who Can Call | Purpose |
|----------|-------------|---------|
| `addCollateral()` | Moderator | Add/update collateral token with rates |
| `addBaseToken()` | Moderator | Add supported lending token |
| `addTerm()` | Moderator | Add supported loan term (min 60 seconds) |
| `setCollateralPrice()` | Oracle Price Feeder or Token Issuer | Update collateral price |
| `update()` | Relayer Owner | Register relayer with lending pairs |
| `updateFee()` | Relayer Owner | Update relayer borrow fee |

---

## 4. Order Lifecycle

```
User signs LendingTransaction
        │
        ▼
  ┌─────────────┐
  │ Lending Pool │  (mempool for lending txs)
  └──────┬──────┘
         │ Block producer picks up pending orders
         ▼
  ┌──────────────────┐
  │ ProcessOrderPending │
  └──────┬───────────┘
         │
    ┌────┴─────┐
    │ ApplyOrder │  (nonce check, verify, route by type)
    └────┬─────┘
         │
    ┌────┴──────────────────────────────────────────────┐
    │                                                    │
    ▼                  ▼              ▼             ▼    ▼
 Limit/Market    Cancel Order      TopUp         Repay  Recall
 (matching)      (refund order)  (add collat.)  (close) (reduce collat.)
    │
    ▼
 Match against opposite side of order book
    │
    ▼
 Create LendingTrade → Settle balances → Update tries
```

### Verification Steps (VerifyLendingItem)

1. **Status validation**: Only `NEW` or `CANCELLED` are valid input statuses
2. **Pair validation**: `(LendingToken, Term)` must be registered on the relayer
3. **Type validation**: Must be one of `LO`, `MO`, `REPAY`, `TOPUP`, `RECALL`
4. **Quantity validation**: Must be > 0
5. **Side validation**: Must be `INVEST` or `BORROW`
6. **Collateral validation** (borrowers only): Must be in the allowed collateral list
7. **Interest validation** (limit orders only): Must be > 0
8. **Relayer validation**: Must be a valid, non-resigned relayer with sufficient deposit
9. **Signature validation**: ECDSA signature must match `userAddress`

---

## 5. Matching Engine

**File**: `tomoxlending/order_processor.go`

The matching engine uses a **double-sided order book** with interest rates as the price axis:

```
     BORROWING SIDE              INVESTING SIDE
  (want lowest rate)          (want highest rate)
  ┌──────────────────┐       ┌──────────────────┐
  │ Interest: 10%    │       │ Interest: 5%     │ ← Best investing rate
  │ Interest: 9%     │       │ Interest: 6%     │
  │ Interest: 8%     │ ← Best│ Interest: 7%     │
  └──────────────────┘ borrow└──────────────────┘

  Match when: borrower's rate >= investor's rate
```

### Limit Order Matching (`processLimitOrder`)

For a **borrower** submitting a limit order at interest rate `I`:
1. Get the lowest investment rate (`minInterest`) from the investing tree
2. While `quantity > 0` AND `I >= minInterest` AND `minInterest > 0`:
   - Match against orders at `minInterest`
   - Create `LendingTrade` for each match
3. If unmatched quantity remains → insert into borrowing tree

For an **investor** submitting a limit order at interest rate `I`:
1. Get the highest borrow rate (`maxInterest`) from the borrowing tree
2. While `quantity > 0` AND `I <= maxInterest` AND `maxInterest > 0`:
   - Match against orders at `maxInterest`
   - Create `LendingTrade` for each match
3. If unmatched quantity remains → insert into investing tree

### Market Order Matching (`processMarketOrder`)

Same as limit order but **no price constraint** — keeps matching until quantity is exhausted or the book is empty.

### Match Settlement per Order (`processOrderList`)

For each match against a maker:
1. Determine the **lesser** of taker quantity and maker amount → `maxTradedQuantity`
2. Get collateral details (depositRate, liquidationRate, recallRate)
3. Get collateral price from oracle/contract/TomoX trading pairs
4. Call `getLendQuantity()` to determine actual tradeable amount based on balances
5. Call `GetSettleBalance()` to compute token transfers
6. Call `DoSettleBalance()` to execute transfers
7. Create `LendingTrade` record
8. Insert into:
   - LendingTrade trie
   - LiquidationTime trie (keyed by `blockTime + term`)
   - LiquidationPrice in TradingState (keyed by liquidation price)

---

## 6. Settlement & Balance Management

**File**: `tomoxlending/lendingstate/settle_balance.go`

### Balance Flow on Match

When a **borrower** matches with an **investor**:

```
BEFORE MATCH:
  Investor: holds [quantityToLend] of LendingToken
  Borrower: holds [collateralAmount] of CollateralToken

AFTER MATCH:
  Investor: LendingToken balance decreases by [quantityToLend]
  Borrower: receives [quantityToLend - borrowFee] of LendingToken
            CollateralToken balance decreases by [collateralLockedAmount]
  Relayer Owner: receives [borrowFee] of LendingToken
  LendingLockAddress: holds [collateralLockedAmount] of CollateralToken (escrow)
  Masternode Owner: receives RelayerLendingFee in TOMO
```

### Collateral Locked Amount Calculation

```
collateralLockedAmount = quantityToLend × collateralTokenDecimal × depositRate
                         ÷ 100 ÷ collateralPrice
```

Example: Borrow 1000 USDT, collateral is ETH
- ETH/USDT price = 2000
- depositRate = 150%
- collateralLockedAmount = 1000 × 10^18 × 150 / 100 / 2000 = 0.75 ETH

### Borrowing Fee

```
borrowFee = quantityToLend × borrowFeeRate / TomoXBaseFee
```

Where `borrowFeeRate` is the relayer's configured fee (e.g., `100 / 10000 = 1%`).

The borrower **receives** `quantityToLend - borrowFee`. The fee goes to the **relayer owner**.

**Investors pay zero fee** in the current design.

---

## 7. Interest Rate Calculation

**File**: `tomoxlending/lendingstate/settle_balance.go`

### Formula

```
InterestRate = APR × (Term + BorrowingTime) / 2 / OneYear

TotalRepayValue = Amount × (BaseInterestDecimal×100 + InterestRate) / (BaseInterestDecimal×100)
```

Where:
- `APR` = Annual Percentage Rate (e.g., `10` = 10%)
- `Term` = loan duration in seconds
- `BorrowingTime` = `currentTime - (liquidationTime - term)` = actual time borrowed
- `OneYear` = 365 days in seconds = `31536000`
- `BaseInterestDecimal` = `10^8` (precision constant)

### Key Insight: Early/Late Repay

The formula `(Term + BorrowingTime) / 2` provides **proportional interest**:
- **Early repay**: `BorrowingTime < Term` → pays **less** interest
- **At expiry**: `BorrowingTime = Term` → pays **full** interest
- **Late repay** (auto-repay after expiry): `BorrowingTime > Term` → pays **more** interest

---

## 8. Collateral & Price Oracle

**File**: `tomoxlending/order_processor.go` — `GetCollateralPrices()`

The system uses a **waterfall price discovery** approach:

```
1. Check LendingRegistration contract for collateralToken/lendingToken price
   (updated by ORACLE_PRICE_FEEDER within current epoch)
   │
   ▼ (if not found)
2. Check contract for inverse price (lendingToken/collateralToken)
   and calculate: collateralPrice = lendTokenDecimal × collateralDecimal / inversePrice
   │
   ▼ (if not found)
3. Check TomoX trading pair: collateralToken/lendingToken
   Use medium trade price from previous epoch
   │
   ▼ (if not found)
4. Calculate from TOMO-denominated prices:
   collateralPrice = collateralTOMOPrice × lendTokenDecimal / lendTokenTOMOPrice
```

### TOMO Base Price Resolution (`GetTOMOBasePrices`)

```
1. If token IS native TOMO → return BasePrice (10^18)
2. Check contract: token/TOMO price → return if fresh
3. Check contract: TOMO/token price → calculate inverse
4. Check TomoX pair: token/TOMO → use medium trade price
```

### Liquidation Price Calculation

```
liquidationPrice = collateralPrice × liquidationRate / depositRate
```

This is the price at which the loan will be liquidated. It's **lower** than the initial collateral price because `liquidationRate < depositRate`.

Example: collateral price = 2000, depositRate = 150, liquidationRate = 110
- liquidationPrice = 2000 × 110 / 150 = 1466.67

---

## 9. Liquidation System

**File**: `tomoxlending/tomoxlending.go` — `ProcessLiquidationData()`

Liquidation runs at every block production. There are **two triggers**:

### 9.1. Liquidation by Time (Loan Expiry)

```go
for lendingBook := range allLendingBooks {
    lowestTime, tradingIds := lendingState.GetLowestLiquidationTime(lendingBook, time)
    for lowestTime > 0 && lowestTime < blockTime {
        for each tradingId:
            ProcessRepayLendingTrade(...)
            // If borrower has enough balance → auto-repay (status: CLOSED)
            // If borrower doesn't have enough → liquidate (status: LIQUIDATED)
    }
}
```

When a loan expires:
1. System attempts **auto-repay**: debit `totalRepayValue` from borrower's lending token balance
2. If borrower **has enough**: loan closes normally (CLOSED), collateral returned
3. If borrower **doesn't have enough**: loan is liquidated

#### Liquidation by Expiry – Collateral Distribution (post-fork)

```
repayAmount = CollateralLockedAmount × LiquidationPrice / currentCollateralPrice + interestInCollateral

If repayAmount < CollateralLockedAmount:
    Borrower gets back: CollateralLockedAmount - repayAmount  (recall)
    Investor gets: repayAmount
Else:
    Investor gets: entire CollateralLockedAmount
```

### 9.2. Liquidation by Price (Collateral Drop)

```go
for each lendingPair (collateralToken, lendingToken):
    collateralPrice = getCurrentPrice(collateralToken, lendingToken)
    // Find all trades where liquidationPrice > currentCollateralPrice
    highestLiquidatePrice, trades = GetHighestLiquidationPriceData(orderbook, collateralPrice)
    for highestLiquidatePrice > 0 && collateralPrice < highestLiquidatePrice:
        for each trade:
            if trade.AutoTopUp:
                AutoTopUp(trade) // try to add more collateral
            else:
                LiquidationTrade(trade) // liquidate
```

When liquidated by price:
- **All** collateral goes to the **investor**
- Borrower loses everything
- Trade status → `LIQUIDATED`
- LiquidationData includes `Reason: "LIQUIDATED_BY_PRICE"`

### 9.3. Auto-Recall (Excess Collateral)

Also runs during `ProcessLiquidationData()`:

```
recallLiquidatePrice = collateralPrice × BaseRecall / recallRate

// Find trades where liquidationPrice < recallLiquidatePrice
// (i.e., the collateral has appreciated significantly)
for each such trade:
    if trade.AutoTopUp:
        ProcessRecallLendingTrade(trade, newLiquidatePrice)
        // Returns excess collateral to borrower
```

This protects borrowers from over-collateralization when their collateral token appreciates.

---

## 10. TopUp / Recall Mechanisms

### 10.1. Manual TopUp

**Triggered by**: Borrower sending a `TOPUP` type LendingTransaction

```go
func ProcessTopUpLendingTrade(lendingTradeId, quantity):
    1. Subtract [quantity] of collateral from borrower
    2. Add [quantity] to LendingLockAddress (escrow)
    3. newLockedAmount = oldLockedAmount + quantity
    4. newLiquidationPrice = oldLiquidationPrice × oldLockedAmount / newLockedAmount
    5. Update trie: liquidation price, locked amount
```

**Effect**: Lowers the liquidation price → makes the loan safer.

### 10.2. Auto TopUp

**Triggered by**: `ProcessLiquidationData()` when `collateralPrice` drops below `liquidationPrice`

```go
func AutoTopUp(lendingBook, lendingTradeId, currentPrice):
    1. newLiquidationPrice = currentPrice × 90% (RateTopUp)
    2. newLockedAmount = oldLockedAmount × oldLiquidationPrice / newLiquidationPrice
    3. requiredDeposit = newLockedAmount - oldLockedAmount
    4. If borrower has enough collateral balance → execute
    5. Else → fall through to liquidation
```

### 10.3. Recall (Collateral Withdrawal)

**Triggered by**: `ProcessLiquidationData()` when collateral has appreciated beyond `recallRate`

```go
func ProcessRecallLendingTrade(lendingBook, lendingTradeId, newLiquidationPrice):
    1. newLockedAmount = oldLockedAmount × oldLiquidationPrice / newLiquidationPrice
    2. recallAmount = oldLockedAmount - newLockedAmount
    3. Return [recallAmount] collateral to borrower
    4. Subtract [recallAmount] from LendingLockAddress
    5. Update liquidation price to newLiquidationPrice (higher = less safe)
```

---

## 11. Repay Flow

**File**: `tomoxlending/order_processor.go` — `ProcessRepayLendingTrade()`

### Normal Repay (Borrower has sufficient balance)

```
1. Calculate totalRepayValue = principal + interest (time-proportional)
2. Subtract totalRepayValue of LendingToken from Borrower
3. Add totalRepayValue of LendingToken to Investor
4. Return CollateralLockedAmount of CollateralToken to Borrower
5. Subtract CollateralLockedAmount from LendingLockAddress
6. Remove from LiquidationTime trie
7. Remove from LiquidationPrice in TradingState
8. Cancel lending trade in LendingState
9. Status → CLOSED
10. ExtraData includes Profit = totalRepayValue - principal
```

### Insufficient Balance at Expiry → Liquidation

If `tokenBalance < paymentBalance` AND `liquidationTime <= currentTime`:
- Pre-fork: entire collateral goes to investor
- Post-fork (`IsTomoXLendingEnabled`): proportional distribution based on `LiquidationExpiredTrade()`

---

## 12. Cancel Flow

**File**: `tomoxlending/order_processor.go` — `ProcessCancelOrder()`

```
1. Verify the order exists in the order book
2. Verify hash matches
3. Verify userAddress matches
4. Check relayer has enough TOMO for cancel fee (RelayerLendingCancelFee)
5. Check user has enough balance to pay token cancel fee
6. Remove order from trie (CancelLendingOrder)
7. Relayer pays TOMO fee to masternode owner
8. User pays token fee to relayer owner:
   - Investor: pays in LendingToken
   - Borrower: pays in CollateralToken
```

### Cancel Fee Calculation

**Pre-fork (getCancelFeeV1)**:
- Investor: `quantity × borrowFeeRate / TomoXBaseCancelFee`
- Borrower: `quantity × collateralTokenDecimal × borrowFeeRate / collateralPrice / TomoXBaseCancelFee`

**Post-fork (getCancelFee)**:
- Converts `RelayerLendingCancelFee` (in TOMO) to the equivalent amount of the relevant token

---

## 13. State Storage (Trie Architecture)

**File**: `tomoxlending/lendingstate/state_lendingbook.go`

Each **Lending Order Book** (identified by `hash(LendingToken, Term)`) maintains **5 separate tries**:

```
lendingExchangeState
├── InvestingTrie        — Orders from investors, keyed by interest rate
│   └── itemListState    — FIFO queue of orders at each rate
│       └── orderItem    — Individual order entries
├── BorrowingTrie        — Orders from borrowers, keyed by interest rate
│   └── itemListState    — FIFO queue of orders at each rate
├── LendingItemTrie      — All orders by LendingId
│   └── lendingItemState — Individual order data
├── LendingTradeTrie     — All active trades by TradeId
│   └── lendingTradeState — Individual trade data
└── LiquidationTimeTrie  — Trades indexed by liquidation timestamp
    └── liquidationTimeState — List of trade IDs at each timestamp
```

Additionally, the **TradingState** (from TomoX spot trading) stores a **LiquidationPrice** index:

```
TradingState
└── OrderBook(collateralToken, lendingToken)
    └── LiquidationPrice trie — Trades indexed by liquidation price
```

### State Root Propagation

The lending state root is stored in a **special transaction** in each block:
- `To`: `common.TradingStateAddr`
- `From`: block author (masternode)
- `Data[0:32]`: Trading state root
- `Data[32:64]`: **Lending state root**

---

## 14. SDK Node Sync & Reorg Handling

**File**: `tomoxlending/tomoxlending.go`

### SyncDataToSDKNode

After matching, SDK nodes (with MongoDB) need to persist results:

1. **Upsert taker order**: Set status (OPEN/FILLED/etc.), update filledAmount
2. **Upsert trades**: Create new LendingTrade records, compute hash
3. **Update maker orders**: Increment filledAmount, update status
4. **Handle rejects**: Mark rejected orders as REJECTED or FILLED (if partially filled)

### RollbackLendingData

On chain reorganization:
1. Restore lending items to their previous state from history cache
2. Restore lending trades to their previous state
3. Delete items/trades that didn't exist before the reorg
4. Remove repay/topup/recall history entries

### History Caches

- `lendingItemHistory`: LRU cache mapping `txHash → {orderHash → LendingItemHistoryItem}`
- `lendingTradeHistory`: LRU cache mapping `txHash → {tradeHash → LendingTradeHistoryItem}`

---

## 15. Fee Structure

| Fee Type | Amount | Paid By | Paid To | Token |
|----------|--------|---------|---------|-------|
| **Borrow Fee** | `quantity × borrowFeeRate / 10000` | Borrower | Relayer Owner | LendingToken |
| **Investing Fee** | 0 (current design) | — | — | — |
| **Matching Fee** | `RelayerLendingFee` (fixed TOMO) | Borrower's Relayer | Masternode Owner | TOMO |
| **Cancel Fee** (relayer) | `RelayerLendingCancelFee` (fixed TOMO) | Relayer | Masternode Owner | TOMO |
| **Cancel Fee** (user) | Based on order size + feeRate | User | Relayer Owner | LendingToken or CollateralToken |

---

## 16. Key Constants & Addresses

| Constant | Description |
|----------|-------------|
| `common.LendingRegistrationSMC` | Address of the LendingRegistration smart contract |
| `common.LendingLockAddress` | Escrow address holding locked collateral |
| `common.RelayerLendingFee` | Fixed TOMO fee per lending match |
| `common.RelayerLendingCancelFee` | Fixed TOMO fee per cancel |
| `common.TomoXBaseFee` | Fee base = 10000 |
| `common.TomoXBaseCancelFee` | Cancel fee base |
| `common.BaseLendingInterest` | Interest precision = 10^8 |
| `common.OneYear` | 365 days in seconds = 31536000 |
| `common.BasePrice` | Price precision = 10^18 |
| `common.RateTopUp` | Auto top-up rate = 90% |
| `common.BaseTopUp` | Top-up base = 100% |
| `common.BaseRecall` | Recall base |
| `DefaultFeeRate` | 100 (1% default in settle_balance.go) |

---

## 17. File Index

### Core Engine

| File | Lines | Purpose |
|------|-------|---------|
| `tomoxlending/tomoxlending.go` | 955 | Main Lending struct, order processing orchestration, SDK sync, reorg, liquidation orchestration |
| `tomoxlending/order_processor.go` | 1218 | Matching engine (limit/market orders), settle balance, cancel, repay, topup, recall, liquidation execution, price calculations |
| `tomoxlending/api.go` | ~80 | RPC API definitions |

### State & Data Structures

| File | Lines | Purpose |
|------|-------|---------|
| `tomoxlending/lendingstate/lendingitem.go` | 465 | LendingItem struct, verification, balance checks |
| `tomoxlending/lendingstate/trade.go` | 191 | LendingTrade struct, BSON serialization |
| `tomoxlending/lendingstate/settle_balance.go` | 257 | Settlement calculation, interest rate formula, repay value |
| `tomoxlending/lendingstate/lendingcontract.go` | 315 | Smart contract state reads (relayer, fee, collateral, terms, pairs) |
| `tomoxlending/lendingstate/state_lendingbook.go` | 793 | Trie-based order book state (5 sub-tries) |
| `tomoxlending/lendingstate/state_lendingitem.go` | ~80 | Single lending item state wrapper |
| `tomoxlending/lendingstate/state_lendingtrade.go` | 79 | Single lending trade state wrapper |

### Transaction Layer

| File | Lines | Purpose |
|------|-------|---------|
| `core/types/lending_transaction.go` | 421 | LendingTransaction type, signing, nonce sorting |
| `core/types/lending_signing.go` | ~100 | Lending transaction signer |
| `core/lending_pool.go` | ~500 | Lending transaction pool (mempool) |
| `core/lending_tx_list.go` | ~300 | Nonce-sorted lending transaction list |

### Smart Contracts

| File | Purpose |
|------|---------|
| `contracts/tomox/contract/LendingRegistration.sol` | On-chain lending governance |
| `contracts/tomox/contract/LendingRegistration.go` | Go bindings |
| `contracts/tomox/lendingRelayerRegistration.go` | Relayer registration bindings |
