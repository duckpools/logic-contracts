# Spectrum Multi-Asset Quote Contract

A logic contract implementation that prices collateral boxes containing ERG plus multiple tokens using Spectrum DEX pools as price oracles.

---

## Required Interface Checklist

| Requirement | Implemented | How |
|-------------|-------------|-----|
| R4[0] borrowLimit | ✓ | Preserved from input (`iBorrowLimit == fBorrowLimit`) |
| R4[1] quotePrice | ✓ | Calculated from DEX reserves |
| R4[2] threshold | ✓ | Calculated as weighted aggregate |
| R4[3] penalty | ✓ | Static value (hardcoded 30) |
| R4[4] minimumValue | ✓ | Preserved from input |
| R4[5] bufferGap | ✓ | Preserved from input |
| R4[6] minimumLoanAmount | ✓ | Preserved from input |
| R4[7] shortLoanFee | ✓ | Preserved from input |
| R4[8] shortLoanDuration | ✓ | Preserved from input |
| R9[0] boxIndex | ✓ | Used to select `boxToQuote` from INPUTS or OUTPUTS |
| Logic NFT in tokens(0) | ✓ | Successor found via `b.tokens(0) == SELF.tokens(0)` |

---

## Overview

This contract prices collateral boxes that may contain **ERG plus multiple tokens**. It uses Spectrum DEX pools as price oracles, converting everything to a final quote in the pool's native currency.

---

## Additional Registers

| Register | Type | Purpose |
|----------|------|---------|
| R5 | `Coll[Coll[Byte]]` | DEX pool NFT IDs. Index 0 = primary (ERG/PoolCurrency), Index 1+ = secondary (Token/ERG) |
| R6 | `Coll[Long]` | Per-asset thresholds. Index 0 = ERG threshold, Index 1+ = token thresholds |
| R7 | `Coll[Long]` | Token amounts from quoted box (ordered to match R5/R6). `0` for assets not present |
| R8 | `Coll[Coll[Byte]]` | Token IDs corresponding to R7 amounts |
| R9 | `Coll[Int]` | `[boxIndex, dexStartIndex]` - extends required R9 with DEX data input location |

---

## Price Calculation Flow

```
┌─────────────────────────────────────────────────────────────┐
│  Quoted Box                                                 │
│  ├── .value (ERG)                                          │
│  └── .tokens(1+) (collateral tokens)                       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 1: Convert each token → ERG via secondary DEX pools  │
│                                                             │
│  For each token with amount > 0:                           │
│    collateralMarketValue = (dexErg * amount * fee) /       │
│      ((dexTokenReserve * 1.02) * 1000 + (amount * fee))    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 2: Sum total ERG value                               │
│                                                             │
│  totalBoxValue = boxERG + convertedTokenERG - networkFee   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 3: Convert total ERG → Pool Currency via primary DEX │
│                                                             │
│  quotePrice = (poolCurrencyReserve * totalERG * fee) /     │
│    ((ergReserve * 1.02) * 1000 + (totalERG * fee))         │
└─────────────────────────────────────────────────────────────┘
```

**Slippage Buffer**: All DEX calculations include a 2% buffer (`Slippage = 2`) to account for price movement between quote creation and transaction confirmation.

---

## Aggregate Threshold Calculation

Each asset type has its own threshold (e.g., ERG might be 800, a volatile token might be 600). The aggregate is weighted by value proportion:

```
aggregateThreshold = Σ (assetValue / totalValue) * assetThreshold
```

Expanded:
```scala
val aggregateThresholdPrimarySum = (boxERG * primaryThreshold) / totalValue

val aggregateThresholdSecondarySum = Σ (tokenERGValue * tokenThreshold) / totalValue

val aggregateThreshold = aggregateThresholdPrimarySum + aggregateThresholdSecondarySum
```

This means a box with mostly ERG gets a threshold close to the ERG threshold, while a box heavy in volatile tokens gets a lower (more conservative) aggregate threshold.

---

## Data Input Requirements

| Index | Box | Validation |
|-------|-----|------------|
| `dexStartIndex` | Primary DEX (ERG/PoolCurrency) | `tokens(0)._1 == primaryDexNft` |
| `dexStartIndex + 1` | Secondary DEX for token 0 | `tokens(0)._1 == secondaryDexNfts(0)` AND `tokens(2)._1 == fOrderedQuotedAssetIds(0)` |
| `dexStartIndex + 2` | Secondary DEX for token 1 | ... |
| ... | ... | ... |

---

## Asset Accounting Validation

The contract ensures every collateral token is properly counted:

1. **`allAssetsCounted`**: Every token in `boxToQuote.tokens.slice(1, ...)` must appear in the `(assetId, amount)` pairs from R7/R8

2. **`correctNumberOfZeroes`**: If R5/R6 are configured for 3 possible tokens but the box only has 1, R7 must have exactly 2 zero entries

3. **`assetsOrderedCorrectly`**: The token ID in each DEX box (`tokens(2)._1`) must match the corresponding entry in R8

---

## Preserved vs Calculated Fields

| Field | Behavior |
|-------|----------|
| borrowLimit | Preserved (admin-set) |
| quotePrice | **Calculated** per quote |
| threshold | **Calculated** as weighted aggregate |
| penalty | Static (hardcoded 30) |
| minimumValue | Preserved (admin-set) |
| bufferGap | Preserved (admin-set) |
| minimumLoanAmount | Preserved (admin-set) |
| shortLoanFee | Preserved (admin-set) |
| shortLoanDuration | Preserved (admin-set) |
| R5 (dexNfts) | Preserved |
| R6 (thresholds) | Preserved |

---

## Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `Slippage` | 2 | 2% price buffer |
| `SlippageDenom` | 100 | Slippage denominator |
| `DexFeeDenom` | 1000 | Spectrum DEX fee denominator |
| `MaximumNetworkFee` | 5,000,000 | 0.005 ERG deducted from collateral value |
| `LargeMultiplier` | 1,000,000,000,000 | Precision for threshold calculation |

---

## Limitations

- **Penalty is static**: Hardcoded to 30, cannot vary per-asset like threshold does
- **Spectrum-specific**: DEX box structure assumes Spectrum format (`tokens(2)` = trading token, `R4[Int]` = fee)
- **Sequential data inputs**: Secondary DEX boxes must be contiguous starting at `dexStartIndex + 1`

---

## Deployments

| Network | Logic NFT | Contract Address |
|---------|-----------|------------------|
| Mainnet | TBD | TBD |
| Testnet | TBD | TBD |
