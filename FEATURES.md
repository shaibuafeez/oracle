# 予測 ORACLE — Feature & Design Brief

A handoff document for redesigning the UI of `predict-market/index.html`. It describes **what the product does, what's on screen today, the data it shows, and every interaction** — enough to design a better interface without reading the code.

> **TL;DR** — A single-page terminal for **DeepBook Predict**, Sui's binary-options prediction market. The user picks a market, drags a strike, reads a live UP/DOWN price, and either paper-trades or signs a **real on-chain mint**. Prices come from a faithful port of the on-chain SVI → N(d₂) pricing engine, fed by the live Sui-testnet indexer.

> **Note:** this brief was written for the original paper-only build. The app has since gained **live testnet data** (a real BTC term structure) and **real on-chain minting** via a connected Sui wallet (`@mysten/sui` + Wallet Standard, loaded lazily on Connect). The UX loop and domain concepts below still hold; see [`README.md`](./README.md) for current status.

---

## 1. What it is

- **Product:** a minimal "options terminal" for predicting whether an asset will be **above or below a chosen strike at expiry**.
- **Domain:** [DeepBook Predict](https://docs.sui.io/onchain-finance/deepbook-predict/) — a prediction market where every strike/expiry is priced off a live **volatility surface**, not fixed odds. Pricing logic is ported from [`@yosuku/deepbook-predict`](https://github.com/yosuku-lab/predict-sdk).
- **Mode:** **paper trade by default; real on-chain mint when a Sui wallet is connected** (builds the SDK's `openUp` PTB and signs it on testnet). Paper positions live in browser memory; on-chain positions carry a tx digest.
- **Data:** **live** — a real BTC term structure pulled from the Sui-testnet indexer (real forwards + on-chain SVI surfaces, polled every 5 s). Seeded surfaces are an offline fallback only.
- **Tech today:** one `index.html` — vanilla JS, inline CSS, hand-rolled SVG. No framework, no build step, no dependencies. A redesign does not have to keep this constraint, but it's why everything is currently in one file.

## 2. Who it's for & the core loop

**User:** someone exploring/learning prediction-market pricing, or demoing DeepBook Predict. Comfortable with crypto, not necessarily with options math.

**The loop the UI must make effortless:**
1. **Pick a market** (BTC / ETH / SOL).
2. **Choose a strike** — "will it be above $X?" — via slider, by dragging the curve, or arrow keys.
3. **Read the price** — the big probability number + UP/DOWN quote tiles update live.
4. **Size & confirm a side** in the ticket (contracts, cost, payout, ROI).
5. **Open a paper position** — it lands in the ledger and **revalues every 1.6 s** as the surface drifts.

The redesign's job: make this loop faster, clearer, and more legible on mobile (the current mobile experience is weak — see §9).

## 3. Domain concepts the UI must communicate

These are the ideas a user has to grasp. The current UI surfaces them with dense labels; a better UI could teach them better.

| Concept | What it means | Where shown today |
|---|---|---|
| **Market / forward** | The asset and its current forward price (≈ spot). | Market list, board header |
| **Strike** | The price threshold the bet is about ("> $63,250?"). | Slider, curve marker, hero question |
| **UP / DOWN** | UP = settles ≥ $1 if price ends **above** strike; DOWN = below. Mutually exclusive sides. | Quote tiles, ticket segmented control, place button |
| **Fair value** | Model probability of UP, in % and as ¢-per-contract (48.7% ≈ $0.487). | Hero number + "fair value" subline |
| **Ask / Bid** | Ask = what you pay to open; Bid = what you'd get to close now. | Quote tiles (ask big, bid small) |
| **Round-trip cost** | Ask − Bid spread; positions open **down** by this amount immediately. | Hero subline; explains why new positions show a small loss |
| **Contract** | 1 contract pays **$1 max** if right. Quantity = number of contracts. | Ticket quantity stepper |
| **Implied vol** | Annualized vol at the strike from the surface. | Board header meta |
| **Expiry / countdown** | When the market settles. | Market list + board header, live ticking |
| **Payout / cost / ROI** | Max payout ($1 × qty), cost (ask × qty), return-if-right. | Ticket summary |
| **SIM vs LIVE** | Whether prices are seeded-simulated or the real on-chain surface. | Status chip in top bar |

## 4. Information architecture (current)

A 3-column "stage" between a top bar and footer:

```
┌─────────────────────────────────────────────────────────────────────┐
│ TOPBAR   予測 ORACLE / predict      [● live/sim] [DeepBook·testnet] 12:04:51 │
├──────────────┬──────────────────────────────────┬────────────────────┤
│  MARKETS     │            BOARD                  │      TICKET        │
│ (248px)      │          (flex 1fr)               │     (312px)        │
│              │                                   │                    │
│ ₿ BTC   54%  │  ₿ Bitcoin      fwd $63,120       │ [Buy Up][Buy Down] │
│   fwd $63k   │                 expires 6d 5h     │                    │
│   6d 5h      │                 impl vol 52%      │ Contracts  − 10 +  │
│              │                                   │                    │
│ Ξ ETH   48%  │  P( BTC > $63,250 at expiry )     │ Entry      48.7¢   │
│   ...        │       54.2 %                      │ Max payout $10.00  │
│              │  fair 54.2¢ · round-trip 2.1¢     │ Profit     $5.13   │
│ ◎ SOL   51%  │                                   │ Cost       $4.87   │
│   ...        │  [== probability curve, SVG ==]   │ Return     105%    │
│              │  Strike ──●────────  $63,250      │                    │
│              │  $53k    at-the-money↑    $73k    │ [ Open Up position]│
│              │                                   │                    │
│              │  ┌── UP ▲ ──┐  ┌── DOWN ▼ ──┐     │ Positions   +$2.40 │
│              │  │ 48.7¢    │  │ 53.4¢      │     │  BTC ▲ UP 10×      │
│              │  │ bid 46.6 │  │ bid 51.3   │     │  +$1.20  ...       │
│              │  └──────────┘  └────────────┘     │                    │
├──────────────┴──────────────────────────────────┴────────────────────┤
│ FOOTER   minimal terminal for DeepBook Predict · engine from predict-sdk │
└─────────────────────────────────────────────────────────────────────┘
```

### Components

**A. Top bar**
- Brand: `予測` (kanji) + `ORACLE / predict`.
- **Data-mode chip** — animated dot + label: `connecting` → `sim · seeded surface` (amber) or `live · on-chain surface` (green).
- Static `DeepBook · testnet` chip.
- Live wall clock (HH:MM:SS).

**B. Markets column (left)**
- List of market cards: glyph (₿ Ξ ◎), ticker, **ATM probability %** (green ≥50, red <50), forward price, expiry countdown.
- Active card has a left UP→DOWN gradient bar + highlighted background.
- Probabilities, forwards, and countdowns tick live.

**C. Board (center) — the focal point**
- **Header:** big glyph, asset name + "TICKER · settles in DUSDC", and right-aligned meta (forward, expiry countdown, implied vol).
- **Hero:** the question `P( BTC > $strike at expiry )`, a huge **probability number** (clamp 72–148px, animated lerp), and a subline with fair-value ¢ and round-trip cost.
- **Probability curve:** full-width SVG of P(up) vs strike across the range. Gradient stroke (green→yellow→red), shaded area, 0/50/100% gridlines, and a draggable **marker dot + line** at the current strike. Animates in by stroke-dash reveal on market switch.
- **Strike control:** label + current `$value`, a range slider with a green→neutral→red track, and a min / "at-the-money ↑" / max ladder.
- **Quote tiles (2-up):** UP and DOWN. Each shows direction label, big **ask** in ¢, a footer with **bid** and "pays $1.00". Selected tile gets a colored inner glow. Clicking a tile selects that side.

**D. Ticket column (right)**
- **Segmented control:** Buy Up / Buy Down (syncs with tile selection; recolors the whole ticket).
- **Quantity stepper:** − / numeric input / + (steps of 5, min 1).
- **Summary rows:** Entry price, Max payout, Profit if right, **Cost** (emphasized), Return if right (ROI %).
- **Place button:** full-width, colored by side ("Open Up position" / "Open Down position"), with a press-bounce animation.
- **Disclaimer note:** explains paper trading + that entry fills the ask, mark uses the bid.
- **Positions ledger:** header with total live PnL (green/red), then position rows (ticker + side + qty, strike, signed PnL, "cost → mark"). New rows slide in. Empty state has guidance copy.

**E. Footer** — attribution links to DeepBook Predict docs and the predict-sdk repo; "paper trading, no funds at risk."

## 5. Interactions

- **Select market** → rebuilds curve (with reveal animation), resets strike to ATM, updates slider bounds/ladder, reprices everything.
- **Strike:** slider drag · **click/drag directly on the curve** · **←/→ arrow keys** nudge by one tick. All snap to the market's tick size.
- **Side:** click a quote tile OR the segmented control — they stay in sync and recolor the place button.
- **Quantity:** ± buttons (step 5) or type (digits only, sanitized).
- **Open position** → prepends to ledger, button bounces.
- **Live heartbeat (every 1.6 s):** SIM markets random-walk their forward; curve, hero, quotes, market list, and all open positions revalue. LIVE market uses real prints instead of drift.
- **Clock/countdowns** tick every 1 s; expiries can reach "settled".
- **Live upgrade:** on load, best-effort fetch of the testnet indexer; on success BTC becomes LIVE and the chip flips green; on failure/timeout it stays SIM (silent).

## 6. States to design for

- **SIM vs LIVE** data mode (chip color + label; today the only at-a-glance signal).
- **Connecting** (brief, on boot).
- **Empty ledger** (guidance copy) vs **populated** (rows + total PnL).
- **Position in profit vs loss** (color, sign).
- **Probability regimes:** near 50% (toss-up) vs deep ITM/OTM (near 1¢/99¢ clamps) — the curve and tiles must stay readable at the extremes.
- **Market expired / "settled"** countdown.
- **Side selected:** UP (green) vs DOWN (red) tints propagate across tiles, segmented control, and place button.

## 7. Current visual identity (keep the soul, upgrade the execution)

A dark, editorial "financial terminal × Japanese minimalism" aesthetic. Worth preserving the character; the redesign can modernize layout, hierarchy, and responsiveness.

- **Palette:** near-black bg (`#09090b`/`#0d0d10`), warm off-white ink (`#ece7db`) with dimmed/faint tiers; **UP green** `#aee05a`, **DOWN red** `#f76d6d`, each with deep + glow variants; amber for SIM.
- **Type:** **Fraunces** (serif display — big numbers, asset names), **JetBrains Mono** (everything else, tabular numerals), and a Japanese serif for `予測`.
- **Texture:** layered radial-gradient atmosphere, a faint dot grid, an animated film-grain overlay, soft-light blends.
- **Motion:** number lerping, stroke-dash curve reveal, pulsing status dot, slide-in ledger rows, panel reveal-on-load stagger, button bounces, hover lifts. *(A redesign exploring richer animation could use the `figma-use-motion` skill if prototyping in Figma.)*
- **Shape language:** rounded cards (11–15px), hairline borders (~10% white), subtle inner glows on active elements.

## 8. Constraints & truths the design must respect

- **Numbers are the product.** Prices, probabilities, PnL — all tabular, all updating live. Legibility and stable layout (no reflow jitter as digits change) are paramount.
- **UP/DOWN is the spine.** Green/red, above/below — this binary must read instantly everywhere.
- **Entry = ask, mark = bid.** New positions correctly open at a small loss (the round-trip spread). The UI must make this *understandable*, not look like a bug.
- **Paper trading.** Never imply real funds or a connected wallet. Keep the "no funds at risk" framing.
- **Live vs simulated** must be honestly and visibly distinguished.
- **DUSDC / testnet / DeepBook** attribution should remain.

## 9. Known weak spots → redesign opportunities

- **Mobile/responsive is thin.** Below 1080px it just stacks the 3 columns; below 560px the quote tiles go 1-up and the brand word hides. There's no real mobile-first layout, no bottom-action-bar pattern for the ticket, no collapsing of the dense board. **Biggest opportunity.**
- **Dense, jargon-heavy labels** ("round-trip cost", "impl vol", "fair value ¢") with no progressive disclosure or tooltips/teaching.
- **The ticket↔board↔tiles relationship** (three places to pick a side, two to read price) can confuse — could be unified.
- **SIM/LIVE distinction** rests on one small chip; easy to miss.
- **Empty/first-run onboarding** is a single line of text — no guided "make your first prediction" moment.
- **Accessibility:** color-only UP/DOWN encoding, custom slider, drag-the-curve, and tiny type need keyboard/contrast/AT review.
- **The probability curve** is beautiful but under-explained — no axis for strike, no hover readout of "P at this price."

## 10. Quick reference — example values

- Markets: **BTC** ₿ (~$63,120, ~6d), **ETH** Ξ (~$3,412, ~2d), **SOL** ◎ (~$146, ~18h).
- A typical quote near ATM: fair **54.2%**, UP ask **48.7¢** / bid 46.6¢, round-trip **~2.1¢**.
- Ticket at 10 contracts: cost **$4.87**, max payout **$10.00**, profit-if-right **$5.13**, ROI **105%**.
- 1 contract = **$1** max payout. Prices quoted in **¢ per contract**; sizes in **contracts**; settlement in **DUSDC**.

---

*Source: `predict-market/index.html` (single-file app). Pricing engine ported from `predict-sdk/src/pricing.ts`. This brief is the design contract — the redesign should preserve the feature set and the truths in §8 while it's free to reinvent layout, hierarchy, responsiveness, and visual execution.*
