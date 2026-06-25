<div align="center">

# 予測 ORACLE

### A minimal, live prediction-market terminal for [DeepBook Predict](https://docs.sui.io/onchain-finance/deepbook-predict/) — Sui's volatility-surface-priced binary options.

**Live BTC term structure · the real SVI → N(d₂) pricing engine · sign a real on-chain mint.**
One self-contained `index.html`. No build step.

</div>

---

## What it is

A single-page "options terminal" for predicting whether BTC settles **above or below a chosen strike at expiry**. You pick a market, drag a strike, read a live UP/DOWN price, and either paper-trade or **mint a real position on Sui testnet**.

Everything is priced by a faithful in-browser port of DeepBook Predict's on-chain pricing — a raw SVI total-variance surface → Black–Scholes digital `N(d₂)` → Bernoulli spread — cross-checked against the contract's own `get_trade_amounts`. The engine is ported from the [`@yosuku/deepbook-predict`](https://github.com/yosuku-lab/predict-sdk) SDK.

## What's real

| Layer | Status |
|---|---|
| **Markets / forwards / surface** | **Live** from the DeepBook Predict testnet indexer — a real BTC **term structure** (≈17 min → ~21 days), polled every 5 s. |
| **Pricing** | **Real** engine — SVI → `N(d₂)` → spread. ATM lands ~48.5%; the book never crosses at the wings. |
| **Order placement** | **Real on-chain mint** when a Sui wallet is connected (`openUp` PTB); **paper trade** otherwise, revaluing against the live bid. |

If the indexer is ever unreachable, the app falls back to a seeded surface so it always renders — clearly badged `seeded · offline`.

## The on-chain mint

Connect a Sui wallet and the order button becomes **Sign & Mint**. It builds the SDK's exact `openUp` PTB and signs it through the Wallet Standard:

```
splitCoins(DUSDC, cost) → predict_manager::deposit
                        → market_key::up|down(oracle, expiry, strike)
                        → predict::mint<DUSDC>(predict, manager, oracle, key, qty, clock)
```

Your `PredictManager` is created automatically the first time. On success the position lands in the ledger with a Suiscan `tx ↗` link, and a toast tracks every step (approve → confirm → minted ✓).

**Prerequisites to mint (testnet):** a Sui wallet (Slush / Sui Wallet) set to **testnet**, and some **DUSDC** from the [faucet](https://tally.so/r/Xx102L). The manager is one extra one-time signature.

`@mysten/sui` + the Wallet Standard load **lazily on first Connect** (from `esm.sh`), so the read-only live app stays dependency-free and works offline against the indexer.

## Run it

It must be served over **HTTP** (a `file://` page sends `Origin: null` and the browser blocks the live fetch):

```bash
cd predict-market
python3 -m http.server 4173
# open http://127.0.0.1:4173
```

That's the whole setup — no install, no bundler.

## Design

A light **editorial "prediction almanac"**: warm paper, ink-black, ledger green / vermilion as the only saturated colors. **Instrument Serif** display over **IBM Plex Mono** data, hairline rules, a dotted-leader order slip, and a probability curve you can **scrub** (slider, drag the curve, or ← → keys).

## Architecture

One `index.html` — vanilla JS, inline CSS, hand-rolled SVG, zero framework:

- **Pricing engine** — `totalVariance` / `normalCdf` / `digitalUp` / `quote`, ported verbatim from the SDK's `pricing.ts`.
- **Live layer** — reads `/oracles`, `/prices/latest`, `/svi/latest`; decodes the sign-magnitude SVI; polls.
- **On-chain layer** — `@mysten/sui` Transaction + Wallet Standard; PTB shape/type-args/arg-order taken from the SDK's `builders.ts` / `keys.ts` / `client.ts`.

See [`FEATURES.md`](./FEATURES.md) for the full UX/interaction brief.

## Credits

- Pricing & PTB shapes: [`@yosuku/deepbook-predict`](https://github.com/yosuku-lab/predict-sdk)
- Protocol: [DeepBook Predict](https://docs.sui.io/onchain-finance/deepbook-predict/) (Sui testnet)

## License

MIT
