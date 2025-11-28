# Logic/Quote Contract Documentation

## Overview

The Logic Contract (also referred to as the Quote Contract) serves as an oracle-like pricing mechanism for the lending protocol. Its primary purpose is to **calculate and report the collateral value** of assets being used to secure loans, enabling the pool and collateral contracts to make informed decisions about borrowing, liquidation thresholds, and loan health.

Any Logic Contract implementation must conform to the interface expected by the Pool and Collateral contracts. Beyond this interface, implementations are free to use any methodology for price discovery and threshold calculation.

---

## Purpose

The Logic Contract fulfills several critical functions:

1. **Price Discovery**: Calculates the market value of collateral assets (methodology is implementation-specific)
2. **Threshold Reporting**: Reports the liquidation threshold applicable to the quoted collateral
3. **Loan Parameter Reporting**: Provides loan settings (limits, fees, durations) to consuming contracts
4. **Box Indexing**: Provides cryptographic linkage between quote outputs and the collateral/loan boxes they reference

---

## Required Interface

**Every Logic Contract implementation MUST provide the following registers, as they are directly read by the Pool and Collateral contracts.**

### R4: Quote Report (`Coll[Long]`) — REQUIRED

A collection of exactly 9 Long values that the Pool and Collateral contracts read by index:

| Index | Field | Description | Referenced By | Valid Range |
|-------|-------|-------------|---------------|-------------|
| 0 | `borrowLimit` | Maximum total borrowed amount allowed from the pool | Pool | > 0 |
| 1 | `quotePrice` | Calculated value of collateral in pool currency | Pool, Collateral | ≥ 0 |
| 2 | `threshold` | Liquidation threshold for this collateral | Pool, Collateral | 1-999 (denominator: 1000) |
| 3 | `penalty` | Liquidation penalty percentage | Collateral | 0-1000 (denominator: 1000) |
| 4 | `minimumValue` | Minimum ERG value required in collateral box | Pool, Collateral | ≥ 1,000,000 |
| 5 | `bufferGap` | Block height gap for liquidation buffer | Pool, Collateral | > 0 |
| 6 | `minimumLoanAmount` | Minimum loan size in pool currency | Pool, Collateral | > 0 |
| 7 | `shortLoanFee` | Fee percentage for early repayment | Pool, Collateral | 0-1000 (denominator: 1000) |
| 8 | `shortLoanDuration` | Block duration where short loan fee applies | Pool, Collateral | ≥ 0 |

### R9: Box Index (`Coll[Int]`) — REQUIRED

At minimum, index 0 must contain the box index value:

| Index | Field | Description |
|-------|-------|-------------|
| 0 | `boxIndex` | Signed index of the collateral/loan box being quoted |

**Why This Is Needed:**

Logic contracts function by quoting a **single box** in a transaction—either an input or an output. Since anyone can use a quote contract to quote any box at any time, the Pool and Collateral contracts must validate that the quoted box in the transaction is actually the relevant one for the operation being validated.

Without this check, an attacker could include a quote for a different, more valuable box and use it to justify an undercollateralized loan.

**Box Index Convention:**

- **Positive values**: Reference OUTPUTS
  - `1` → `OUTPUTS(0)`
  - `2` → `OUTPUTS(1)`
  - `n` → `OUTPUTS(n-1)`
  
- **Negative values**: Reference INPUTS
  - `-1` → `INPUTS(0)`
  - `-2` → `INPUTS(1)`
  - `-n` → `INPUTS(n-1)`

**Validation in Pool Contract (borrow):**
```scala
val collateralBox = plausibleCollateralBoxes(0)
val collateralIndex = OUTPUTS.map{
    (b: Box) => b.id
}.indexOf(collateralBox.id, 0)

// ... later ...

val isQuotedBoxValid = collateralIndex == fQuote.R9[Coll[Int]].get(0) - 1
```

**Validation in Collateral Contract (partial repay, adjust collateral):**
```scala
val fCollateral = fCollaterals.getOrElse(0, SELF)
val collateralIndex = OUTPUTS.map{
    (b: Box) => b.id
}.indexOf(fCollateral.id, 0)
val isQuotedBoxValid = collateralIndex == fQuote.R9[Coll[Int]].get(0) - 1
```

**Validation in Collateral Contract (liquidation):**
```scala
val collateralIndex = INPUTS.map{
    (b: Box) => b.id
}.indexOf(SELF.id, 0)
val isQuotedBoxValid = collateralIndex == fQuote.R9[Coll[Int]].get(0) * -1 - 1
```

---

## How Logic Contracts Are Located

### Token Identification

Logic contracts are identified by their first token (NFT). Valid Logic NFTs are registered in the Pool's parameter box:

```scala
val poolSettings = CONTEXT.dataInputs.filter{...}(0)
val customLogicNFTs = poolSettings.R4[Coll[Coll[Byte]]].get
```

### From Pool Contract (Borrow Operations)

The pool locates the quote box by matching against registered Logic NFTs AND the collateral's specified quote NFT:

```scala
val fQuote = OUTPUTS.filter{
    (b: Box) => b.tokens.size > 0 && customLogicNFTs.exists{
        (NFT: Coll[Byte]) => b.tokens(0)._1 == NFT && NFT == collateralQuoteNFT
    }
}(0)
```

### From Collateral Contract (All Operations)

The collateral contract stores its quote NFT in R7 and uses it to locate quote boxes:

```scala
val currentQuoteNFT = SELF.R7[Coll[Byte]].get

val fQuotes = OUTPUTS.filter{
    (b: Box) => b.tokens.size > 0 && b.tokens(0)._1 == currentQuoteNFT
}
```

---

## How R4 Fields Are Used

### Pool Contract Usage

During **borrow operations**, the pool reads:

| Field | Usage |
|-------|-------|
| `borrowLimit` (0) | Validates `finalBorrowedFromPool < borrowLimit` |
| `quotePrice` (1) | Validates `quotePrice >= loanAmount * threshold / 1000` |
| `threshold` (2) | Used in collateral sufficiency check |
| `penalty` (3) | Recorded in collateral box R9 |
| `minimumValue` (4) | Validates `collateralValue >= minimumValue` |
| `bufferGap` (5) | Recorded in collateral box R9 |
| `minimumLoanAmount` (6) | Validates `loanAmount >= minimumLoanAmount` |
| `shortLoanFee` (7) | Recorded in collateral box R9 |
| `shortLoanDuration` (8) | Recorded in collateral box R9 |

### Collateral Contract Usage

During **partial repayment**:

| Field | Usage |
|-------|-------|
| `quotePrice` (1) | Validates remaining collateral sufficiency |
| `threshold` (2) | Used in `quotePrice >= remainingDebt * threshold / 1000` |

During **liquidation preparation** (readyToLiquidate):

| Field | Usage |
|-------|-------|
| `quotePrice` (1) | Checks if loan is underwater |
| `threshold` (2) | Used in undercollateralization check |

During **liquidation execution**:

| Field | Usage |
|-------|-------|
| `quotePrice` (1) | Calculates liquidation proceeds and borrower share |
| `threshold` (2) | Confirms liquidation is valid |
| `penalty` (3) | Calculates penalty distribution |

During **collateral adjustment**:

| Field | Usage |
|-------|-------|
| `quotePrice` (1) | Validates new collateral is sufficient |
| `threshold` (2) | New threshold to record in collateral |
| `penalty` (3) | New penalty to record in collateral |

---

## Constants Used by Consuming Contracts

These constants are defined in the Pool and Collateral contracts and affect how R4 values are interpreted:

| Constant | Value | Used For |
|----------|-------|----------|
| `LiquidationThresholdDenomination` | 1000 | Denominator for threshold (R4[2]) |
| `PenaltyDenom` | 1000 | Denominator for penalty (R4[3]) |
| `MinimumBoxValue` | 1,000,000 | Minimum ERG (affects R4[4] minimum) |
| `defaultBufferHeight` | 100,000,000 | Default buffer when loan is healthy |

---

## Security Requirements

Any Logic Contract implementation must ensure:

1. **Unique Logic NFT**: The Logic NFT must have exactly 1 token minted, and the logic contract box must hold this NFT. This ensures quotes cannot be forged by creating duplicate NFTs.

2. **Accurate Box Indexing**: R9[0] must correctly identify the box being quoted. Incorrect indexing could allow quote reuse attacks.

3. **Parameter Bounds**: All R4 values should stay within valid ranges to prevent overflow/underflow in consuming contracts.

---

## Implementation Checklist

When creating a new Logic Contract:

- [ ] Output box has unique identifying NFT as `tokens(0)`
- [ ] NFT is registered in parameter box `R4[Coll[Coll[Byte]]]`
- [ ] R4 contains exactly 9 Long values
- [ ] R4[0] (borrowLimit) reflects appropriate pool limits
- [ ] R4[1] (quotePrice) accurately reflects collateral value in pool currency
- [ ] R4[2] (threshold) is between 1-999
- [ ] R4[3] (penalty) is between 0-1000
- [ ] R4[4] (minimumValue) is ≥ 1,000,000
- [ ] R4[5] (bufferGap) provides adequate liquidation warning
- [ ] R4[6] (minimumLoanAmount) prevents dust loans
- [ ] R4[7] (shortLoanFee) is between 0-1000
- [ ] R4[8] (shortLoanDuration) is reasonable block count
- [ ] R9[0] correctly indexes the quoted box (positive for OUTPUTS, negative for INPUTS)

---

---

# Example Implementation: DEX-Based Logic Contract

The following section describes the **specific implementation** provided in `generate_logic_script()`. This is one possible approach; other implementations could use different price sources (oracles, different DEX protocols, etc.).

## Overview

This example implementation calculates collateral value using on-chain Spectrum DEX liquidity pools. It:

1. Reads ERG value directly from the collateral box
2. Converts additional collateral tokens to ERG via DEX reserves
3. Converts total ERG value to pool currency via primary DEX
4. Calculates weighted aggregate threshold based on asset composition

## Additional Registers (Implementation-Specific)

Beyond the required R4 and R9, this implementation uses:

### R5: DEX NFT Configuration (`Coll[Coll[Byte]]`)

| Index | Purpose |
|-------|---------|
| 0 | Primary DEX NFT (ERG/PoolCurrency pair) |
| 1+ | Secondary DEX NFTs (for collateral token → ERG conversion) |

### R6: Asset Thresholds (`Coll[Long]`)

Per-asset liquidation thresholds used to calculate weighted aggregate:

| Index | Purpose |
|-------|---------|
| 0 | ERG threshold |
| 1+ | Secondary token thresholds |

### R7: Ordered Asset Amounts (`Coll[Long]`)

Token amounts from the collateral box, ordered to match R5/R6:
- `0` value indicates token not present in this collateral

### R8: Ordered Asset IDs (`Coll[Coll[Byte]]`)

Token IDs corresponding to R7 amounts (for validation).

### R9: Extended Helper Indices (`Coll[Int]`)

| Index | Purpose |
|-------|---------|
| 0 | Box index (REQUIRED) |
| 1 | DEX start index in data inputs |

## Price Calculation

### Token → ERG Conversion

```scala
val collateralMarketValue = (dexReservesErg * inputAmount * dexFee) /
    ((dexReservesToken._2 + (dexReservesToken._2 * Slippage / 100)) * DexFeeDenom +
    (inputAmount * dexFee))
```

### Total ERG → Pool Currency

```scala
val quotePrice = (yAssets * totalBoxValue * dexFee) /
    ((xAssets + (xAssets * Slippage / SlippageDenom)) * DexFeeDenom +
    (totalBoxValue * dexFee))
```

Where:
- `xAssets` = DEX ERG reserves
- `yAssets` = DEX pool currency reserves
- `Slippage` = 2% buffer
- `totalBoxValue` = ERG in collateral + converted token values - network fee

### Aggregate Threshold

Weighted by each asset's proportion of total value:

```scala
val aggregateThreshold = (
    (ergValue * primaryThreshold / totalValue) +
    Σ(tokenValue * tokenThreshold / totalValue)
)
```

## Constants (This Implementation)

| Constant | Value | Purpose |
|----------|-------|---------|
| `Slippage` | 2 | 2% price buffer |
| `SlippageDenom` | 100 | Slippage denominator |
| `DexFeeDenom` | 1000 | DEX fee denominator |
| `MaximumNetworkFee` | 5,000,000 | Deducted from collateral value |
| `LargeMultiplier` | 1,000,000,000,000 | Precision for threshold math |

## State Transition Rules

This implementation enforces that settings remain constant between states:

```scala
val scriptRetained = outLogic.propositionBytes == SELF.propositionBytes
val quoteSettingsRetained = fDexNfts == iDexNfts && fAssetThresholds == iAssetThresholds
val iBorrowLimit == fBorrowLimit
val iMinimumValue == fMinimumValue
val iBufferGap == fBufferGap
val iMinimumLoanAmount == fMinimumLoanAmount
val iShortLoanFee == fShortLoanFee
val iShortLoanDuration == fShortLoanDuration
```

Only `quotePrice`, `threshold` (aggregate), and `penalty` change between quotes.

## Data Input Requirements

| Index | Box | Required Tokens/Registers |
|-------|-----|---------------------------|
| `dexStartIndex` | Primary DEX | `tokens(0)` = DEX NFT, `tokens(2)` = pool currency, `R4[Int]` = fee |
| `dexStartIndex + 1` to `n` | Secondary DEXs | `tokens(0)` = DEX NFT, `tokens(2)` = collateral token, `R4[Int]` = fee |

## Validation Rules

| Rule | Purpose |
|------|---------|
| `allAssetsCounted` | Every collateral token must appear in R7/R8 |
| `assetsOrderedCorrectly` | R8 token IDs must match DEX pool tokens |
| `dInsMatchesAssetsSize` | Number of DEX data inputs matches asset count |
| `correctNumberOfZeroes` | Missing assets properly marked as 0 |
| `isValidPrimaryDexBox` | Primary DEX NFT matches configuration |
