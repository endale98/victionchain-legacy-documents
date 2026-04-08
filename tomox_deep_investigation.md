# TomoX Trading & TomoX Lending: Deep Investigation Report

This report provides a comprehensive technical overview of TomoX (DEX) and TomoZ (Lending) implementations in the Viction (formerly TomoChain) ecosystem, based on the `victionchain` (legacy) and `vic-geth` (modern) codebases.

---

## 1. Architectural Overview

TomoX and TomoZ are built as "layer 1.5" protocols. They are integrated directly into the Viction blockchain core but maintain their own state separate from the standard EVM state (though they interact with it).

### Key Components:
- **OrderTransaction (OrderTx)**: A specialized transaction type for submitting, updating, or cancelling orders. It is distinct from standard Ethereum transactions.
- **OrderPool**: A dedicated pool for `OrderTransaction`s, similar to the standard `TxPool`. It performs protocol-specific validation (e.g., checking if the user has enough locked balance).
- **TradingStateDB**: A separate Merkle Patricia Trie that stores the entire TomoX order book state.
- **LendingStateDB**: A separate Merkle Patricia Trie that stores the TomoZ lending state.
- **System Transactions**: Specialized standard transactions (identified by specific 'To' addresses) used by masternodes to commit matching results and state roots to the blockchain.

---

## 2. Data Structures & State Management

### State Database Hierarchy
Unlike the EVM state which maps `Address -> Account`, the TomoX/TomoZ states are hierarchical.

#### TradingStateDB (TomoX)
- **Main Trie**: Maps `OrderBookHash` (Hash of BaseToken + QuoteToken) -> `tradingExchangeObject`.
- **tradingExchangeObject**:
    - `AskRoot`: Sub-trie for Ask (Sell) orders, indexed by Price.
    - `BidRoot`: Sub-trie for Bid (Buy) orders, indexed by Price.
    - `OrderRoot`: Sub-trie mapping `OrderID` -> `OrderItem` details.
    - `LiquidationPriceRoot`: Sub-trie for tracking liquidation prices (integrated with Lending).
    - `LastPrice`, `MediumPrice`, `MediumPriceBeforeEpoch`: Market price tracking.

#### LendingStateDB (TomoZ)
- Similar structure, mapping `LendingOrderBookHash` (LendingToken + Term) -> `lendingExchangeState`.
- Tracks `Investing` (Lending) and `Borrowing` order books.
- Manages `LendingTrade` objects for active loans.

### What is committed to the Trie?
- Every order submission, match, or cancellation updates the respective sub-trie.
- The `TradingStateDB` root and `LendingStateDB` root are calculated at the end of every block.

### What is written to the DB?
- The trie nodes (standard LevelDB storage for tries).
- The matching results (for SDK nodes to query history).

---

## 3. Transaction Flow & Lifecycle

### Stage 1: Order Submission
1. User creates and signs an `OrderTransaction`.
2. The transaction is broadcast to the network.
3. Masternodes receive it in their `OrderPool`.
4. **Validation**: The `OrderPool` checks:
    - Signature validity.
    - Nonce (managed in `TradingStateDB`).
    - Balance: Checks if the user has enough tokens/VIC. Note that TomoX uses a "locking" mechanism where balance is checked against `statedb` but considered "reserved" by the protocol.

### Stage 2: Matching (Miner/Masternode Side)
1. During block production (in `miner/worker.go`), the masternode creator calls `ProcessOrderPending`.
2. The matching engine looks for overlapping Buy/Sell or Invest/Borrow orders.
3. **Matching Engine**:
    - Executes matching logic (Price-Time priority).
    - Calculates trades.
    - Updates local `TradingStateDB` / `LendingStateDB` copies.
4. **System Transaction Creation**:
    - **0x91**: If matches are found, the miner creates a transaction to `0x...91` containing the RLP-encoded `TxMatchBatch` (all matched trades in this block).
    - **0x93**: Similar for TomoZ lending matches.
    - **0x94**: For finalized lending trades (liquidations).

### Stage 3: State Root Commitment (The 0x92 Transaction)
1. After all matching is done, the miner calculates the new `TradingStateDB` and `LendingStateDB` root hashes.
2. The miner creates a system transaction to `TradingStateAddr` (**0x92**).
3. **Payload**: 64 bytes containing `[32 bytes TradingRoot | 32 bytes LendingRoot]`.
4. This transaction is included in the block to provide a consensus-verifiable state root for the DEX/Lending states.

### Stage 4: Block Processing & Settlement (Full Node Side)
1. When any node processes the block (`StateProcessor.Process`):
2. It encounters the **0x91/0x93** transactions.
3. Instead of running EVM code, it triggers `applyTomoXTx` / `applyLendingTx`.
4. These functions:
    - Decode the match batch.
    - Replay the trades through `CommitOrder`.
    - **EVM Settlement**: Updates `statedb` to transfer token balances between users (Buyer -> Seller, etc.).
    - **Protocol Settlement**: Updates `TradingStateDB` / `LendingStateDB` (reducing order quantities, closing orders).
5. It encounters the **0x92** transaction and records the expected state roots.
6. **Finalization**: In `afterProcess`, the node commits its local TomoX/Lending tries and **verifies** that its calculated roots match the ones in the **0x92** transaction. If they don't, the block is invalid.

---

## 4. Hook Architecture in `vic-geth`

In the modern `vic-geth` client, this logic is isolated via hooks to maintain upstream compatibility:

| Hook | Action |
|---|---|
| `beforeProcess` | Opens the tries from the parent block; updates medium prices at epoch boundaries. |
| `applyVictionTransaction` | Detects **0x91-0x94** transactions. Executes the matching replay and EVM balance updates. |
| `afterProcess` | Commits the tries to get the new roots; verifies them against the **0x92** transaction data. |

---

## 5. Technical Details & Constants

### System Addresses
- `0x88`: Masternode Voting SMC
- `0x89`: Block Signers tracking (Deleted after TIPSigning)
- `0x90`: Randomize SMC
- **0x91**: TomoX Match Batch Address
- **0x92**: Trading State Root Commitment Address
- **0x93**: TomoZ Lending Match Batch Address
- **0x94**: TomoZ Finalized Trade Address

### Database Keys
- The tries are stored in LevelDB.
- `TradingStateDB` uses a dedicated `trie.Database`.
- Matching results are often stored in a separate bucket for fast RPC retrieval.

### Price Calculation
- Prices are handled as `*big.Int` to maintain precision.
- "Medium Price" is updated every epoch (900 blocks) to prevent price manipulation and provide a reference for liquidations.

---

## 6. Summary of Stages

| Stage | Actor | State Changes | Transaction Type |
|---|---|---|---|
| **Submission** | User | None (queued) | `OrderTransaction` |
| **Matching** | Miner | Local Tries Updated | Internal Logic |
| **Commitment** | Miner | None | **0x91/0x93** (Matches), **0x92** (Roots) |
| **Settlement** | All Nodes | **EVM Balances** + **Order Tries** | Replay of **0x91/0x93** |
| **Verification** | All Nodes | Trie Flush to Disk | Verification of **0x92** |

This architecture ensures that while the DEX and Lending logic are complex and "off-EVM", their results are fully settled on-chain with the same security guarantees as standard transactions.