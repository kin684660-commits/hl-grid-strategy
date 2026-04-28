# Hyperliquid Grid Trading Strategy

skill_id: hl-grid-strategy
basic_skills: [hyperliquid-plugin]

## Overview

This skill runs an automated grid trading strategy on Hyperliquid perpetual futures using the hyperliquid-plugin. It places multiple limit buy and sell orders across a defined price range, profiting from price oscillations while accumulating trading volume.

## Strategy Logic

1. Fetch current BTC-USDC price from Hyperliquid
2. Calculate grid levels above and below current price
3. Place limit BUY orders below current price (maker orders, 0.015% fee)
4. Place limit SELL orders above current price (maker orders, 0.015% fee)
5. When a BUY fills → immediately place a SELL one grid level higher
6. When a SELL fills → immediately place a BUY one grid level lower
7. Repeat continuously within the configured range

## Default Parameters

- Asset: BTC-USDC perpetual
- Grid count: 10 levels (5 buy + 5 sell)
- Grid spacing: 0.3% between each level
- Order size per grid: $3 notional (adjustable)
- Leverage: 3x
- Order type: Limit only (maker, to minimize fees)

## Pre-flight Checklist

Before running, verify:
- [ ] onchainos CLI installed and configured with OKX API keys
- [ ] hyperliquid-plugin installed
- [ ] Agentic Wallet funded with USDC (minimum $30 recommended)
- [ ] Hyperliquid-plugin and Onchain OS updated to latest version

## Commands

### Step 1 — Check wallet balance
```
onchainos wallet balance
```

### Step 2 — Get current BTC price on Hyperliquid
```
onchainos dapp hyperliquid-plugin price --asset BTC
```

### Step 3 — Calculate grid levels
Given current price P, grid spacing G = 0.3%:
- Buy levels: P×(1-G), P×(1-2G), P×(1-3G), P×(1-4G), P×(1-5G)
- Sell levels: P×(1+G), P×(1+2G), P×(1+3G), P×(1+4G), P×(1+5G)

### Step 4 — Place grid orders
For each buy level:
```
onchainos dapp hyperliquid-plugin order --side buy --type limit --asset BTC --price <level_price> --size <order_size> --leverage 3
```

For each sell level:
```
onchainos dapp hyperliquid-plugin order --side sell --type limit --asset BTC --price <level_price> --size <order_size> --leverage 3
```

### Step 5 — Monitor and refill orders
Check open orders every 5 minutes:
```
onchainos dapp hyperliquid-plugin orders --asset BTC
```

When a filled order is detected, place the corresponding counter-order one grid level away.

### Step 6 — Cancel all orders (stop strategy)
```
onchainos dapp hyperliquid-plugin cancel-all --asset BTC
```

## Risk Warning

- Grid trading does not guarantee profit
- Trending markets can cause one-sided fills and inventory risk
- Always use funds you can afford to lose
- Monitor positions regularly
- This skill is for educational and competition purposes

## Fee Estimation

All orders use limit (maker) type → fee = 0.015% per fill
- 10 fills × $3 notional × 0.015% = ~$0.0045 in fees per full cycle
- Budget of $30 can sustain hundreds of cycles
