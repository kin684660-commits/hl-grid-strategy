# HL Grid Strategy

**Automated grid trading on Hyperliquid perpetuals via OKX Plugin Store**

This strategy places a ladder of limit buy and sell orders around the current BTC price on Hyperliquid, automatically refilling orders as they fill. It is designed to generate high trade counts while keeping fees minimal by using maker-only limit orders (0.015% fee rate).

## Key Features

- Maker-only orders to minimize fees (0.015% vs 0.045% taker)
- Automatic order refill on each fill
- Configurable grid spacing and order size
- Built on top of `hyperliquid-plugin` from OKX Plugin Store
- Compatible with OKX Agentic Wallet

## Risk Level: High

Perpetual futures trading involves significant risk of loss.
