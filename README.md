# 🌐 Solana Volume Bot Multi-DEX Anatomy — How a Pump.fun Volume Bot Routes Across Six Venues

> A modern Solana token launch is not a single-venue event. It begins on Pump.fun, often crosses through Bonk.fun, graduates to Raydium, distributes depth into Orca, is routed against Jupiter, and is discovered through DexScreener. A Solana volume bot built for one of those venues is silently failing in the other five. This guide walks through the six venues, explains how a multi-DEX Pump.fun volume bot coordinates execution across all of them, and uses [**PumpBot**](https://www.pumpbot.net/) as the working reference implementation.

---

## 🗺️ The Six Venues That Define a Solana Token Launch

Every Solana token launch in the current market lives across six interconnected venues, and the way capital, attention, and price discovery flow between them is what separates a launch that surfaces from a launch that does not. A modern Pump.fun volume bot needs venue-specific logic for all six — not generic routing, not Ethereum-ported templates, not "good enough" approximations. You can see the full multi-DEX architecture documented at [**https://www.pumpbot.net**](https://www.pumpbot.net/), which is the platform this guide unpacks venue by venue.

The six venues, in the order a typical token touches them:

| Venue | Role | What the Bot Must Know |
|---|---|---|
| 🚀 **Pump.fun** | Primary bonding-curve launchpad | Native curve calls, trending thresholds, comment + favorite signals |
| 🐶 **Bonk.fun** | Alternative launchpad | Bonk-tier trending, first-block buy pressure, native chat UI |
| 💧 **Raydium** | Post-migration AMM | CLMM + v4 routing, tick-aware slippage, block-by-block migration detection |
| 🌊 **Orca** | Whirlpools concentrated liquidity | Tick-aware execution, range-aware sizing |
| 🪐 **Jupiter** | Aggregator | Best-route resolution, multi-hop awareness, per-swap consultation |
| 📊 **DexScreener** | Discovery + trending surface | Refresh-cycle awareness, minute-edge spike timing |

A Solana volume bot that ignores any one of these surfaces is a Pump.fun volume bot in name only. The rest of this guide walks through how a multi-DEX engine actually handles each.

---

## 🚀 Pump.fun — The Entry-Point Launchpad

Pump.fun is the dominant entry surface for Solana launches and the venue where most Pump.fun volume bot sessions begin. The mechanics are bonding-curve based: each token has a deterministic price function that maps SOL committed to tokens received, and graduation to Raydium happens when the curve reaches a fixed market-cap threshold.

What a Pump.fun-aware Solana volume bot does at this venue:

- 📈 **Native bonding-curve routing.** Orders go through Pump.fun's program directly, not through a wrapper. Curve-cap detection is automatic.
- 🎯 **Trending threshold targeting.** Volume, holder count, comment activity, and favorite count are sampled by the trending algorithm in concert; the bot fires them together to cross the thresholds at the same moment.
- 💬 **First-block comment deployment.** Comments and favorites land within the first blocks of bot activity, not after, so the social signal builds in parallel with the volume signal.
- ⚡ **First-block buy pressure.** Aggressive buy-side flow in the opening window when the trending algorithm samples most heavily.

The Pump.fun stage is where a Solana volume bot earns or loses its first opportunity. A session that handles bonding-curve mechanics correctly surfaces; a session that treats Pump.fun like a generic AMM mis-routes from the first fill.

---

## 🐶 Bonk.fun — The Bonk-Ecosystem Launchpad

Bonk.fun is the second credible launchpad in the Solana ecosystem and the routing target for tokens minted under the Bonk umbrella. The mechanics are similar to Pump.fun — bonding curve, trending feed, graduation threshold — but the audience, the trending algorithm's sampling, and the native chat UI differ in ways a multi-DEX Pump.fun volume bot has to account for.

What a Bonk.fun-aware Solana volume bot does:

- 🐶 **Bonk-tier trending push.** Bonk.fun's trending surface weights volume and holder count slightly differently than Pump.fun; the bot's curve shape adjusts.
- 💬 **Native chat UI auto-comments and favorites.** Comments deploy through the Bonk.fun-specific chat surface, not the Pump.fun chat surface — the wire format is different and a bot built only for Pump.fun fails silently here.
- ⚡ **First-block buy pressure tuned to Bonk audience.** The community on Bonk.fun samples differently from the cross-launchpad audience on Pump.fun.

A Solana volume bot that treats Bonk.fun as a clone of Pump.fun is leaving the second launchpad on the table. A multi-DEX Pump.fun volume bot like [**PumpBot**](https://www.pumpbot.net/) handles both natively.

---

## 💧 Raydium — The Migration Target AMM

When a token graduates from Pump.fun or Bonk.fun's bonding curve, it migrates to Raydium — the dominant Solana AMM. Raydium serves two pool types that a Pump.fun volume bot must handle differently:

| Pool Type | Mechanics | Routing |
|---|---|---|
| **CLMM** (Concentrated Liquidity) | Tick-based positions like Uniswap v3 | Tick-aware slippage calculation, range checks |
| **Standard v4** | Constant-product like Uniswap v2 | Standard AMM math |

The Raydium routing layer is where the highest-stakes failure mode for a Solana volume bot lives: the **migration handoff**. The moment the bonding curve graduates, routing must switch from the launchpad's native program to the appropriate Raydium pool — block-by-block detection, no paused session, no dropped volume. A bot that misses the handoff window hands the trending slot to whichever launch is next in queue.

What a Raydium-aware Pump.fun volume bot does:

- 🔄 **Block-by-block migration detection.** Monitors graduation status every block, switches routing target the moment migration confirms.
- 🎯 **CLMM tick-aware sizing.** Concentrated-liquidity positions have non-linear price impact near tick boundaries; slippage calculation accounts for it.
- 💧 **v4 standard routing.** Falls back to constant-product math for tokens migrated to v4 pools.
- ⏸️ **Zero session pause.** The migration boundary is the visibility peak of the launch — pausing the session there is the single most expensive operational error a Solana volume bot can make.

---

## 🌊 Orca — Concentrated Liquidity Whirlpools

Orca's Whirlpools are Solana's leading concentrated-liquidity venue and a common cross-DEX mirroring target for the post-migration phase of a launch. A multi-DEX Pump.fun volume bot uses Orca to distribute depth: simultaneous activity on Raydium and Orca produces a token whose emerging price discovery is not concentrated on a single pool.

What an Orca-aware Solana volume bot does:

- 🌊 **Whirlpool tick logic.** Concentrated-liquidity execution respects active tick ranges; orders that would land outside the range are skipped or adjusted.
- 📏 **Range-aware sizing.** Position sizing accounts for the tick range the liquidity actually occupies, not the theoretical maximum.
- 🎯 **Tick-aware slippage tuning.** Real-time slippage estimation from current pool depth at the active tick.

Cross-DEX mirroring across Raydium and Orca during the post-migration phase is one of the practical differentiators between a Pump.fun volume bot built for single-venue execution and one engineered for the full Solana launch lifecycle.

---

## 🪐 Jupiter — The Aggregator That Picks the Best Route

Jupiter is the dominant Solana DEX aggregator. Almost every meaningful Solana swap consults Jupiter for best-price routing. A modern Pump.fun volume bot integrates Jupiter at every swap — not as a fallback, but as the default consultation step.

What a Jupiter-aware Solana volume bot does:

- 🪐 **Per-swap best-route resolution.** Every fill consults Jupiter for the cheapest multi-hop route; if a direct pool fill is cheaper, it falls back to direct routing.
- 🔀 **Multi-hop awareness.** Jupiter routes that span multiple pools are executed as atomic transactions, not sequential trades.
- 🔄 **Automatic failover.** If a Jupiter route fails, the engine falls back to the next-best route, then to direct pool routing, then surfaces the failure.

The Jupiter integration is mostly invisible from the operator's perspective — which is exactly the point. A Solana volume bot that surfaces routing decisions to the operator is asking the operator to do work the engine should be doing.

---

## 📊 DexScreener — The Discovery Surface

DexScreener is not a DEX. It is the trending and pair-discovery surface where retail traders actually find new tokens. A Pump.fun volume bot that does not understand DexScreener is producing volume nobody is watching for.

What a DexScreener-aware Solana volume bot does:

- ⏰ **Refresh-cycle burst timing.** DexScreener's trending algorithm samples at minute-edge intervals (`:00`, `:15`, `:30`, `:45` seconds). The bot's burst-mode fires precisely at those moments so the volume spike is observed when the algorithm looks.
- 🔥 **Hot-pair signal shaping.** Volume curve shape is tuned to match the patterns DexScreener's hot-pair detection prefers — sustained presence with calibrated bursts, not pure spike-and-fade.
- 📊 **Minute-edge spike injection.** Smaller spikes timed precisely to sampling windows produce disproportionate visibility versus the same volume distributed uniformly.

DexScreener is the surface where retail discovery actually happens for the majority of Solana launches. A Pump.fun volume bot that ignores its sampling cadence is producing volume against a clock the discovery audience is not watching.

---

## ⚙️ How a Multi-DEX Solana Volume Bot Coordinates All Six

The hard problem is not handling any single venue correctly — it is handling all six in coordination, in real time, while routing changes as the token migrates through its lifecycle. A multi-DEX Pump.fun volume bot does this through a routing layer that:

1. 🎯 **Detects the active venue.** On every fill, the engine determines whether the token is on a Pump.fun bonding curve, a Bonk.fun curve, a Raydium CLMM pool, a Raydium v4 pool, or an Orca Whirlpool.
2. 🪐 **Consults Jupiter.** For every active-venue route, the engine compares Jupiter's best multi-hop price against the direct pool quote and picks the cheaper option.
3. 📊 **Times against DexScreener sampling.** Burst-mode spikes are scheduled to land at minute-edge windows.
4. 🔄 **Watches for migration.** The bonding-curve monitor runs block-by-block; the moment migration lands, the active-venue detection switches automatically.
5. 🌊 **Mirrors cross-DEX.** Optional Meteora and Orca mirroring runs simultaneous activity during the post-migration phase to distribute emerging depth.

The result is a single Solana volume bot session that produces venue-correct execution across the entire Solana launch lifecycle — bonding curve → graduation → AMM depth → aggregator routing → discovery surface — without operator intervention at any stage transition. The reference [**Pump.fun Volume Bot**](https://www.pumpbot.net/) implementing this coordination ships it as the default routing path, not a premium tier.

---

## 🔐 Non-Custodial Architecture

A Pump.fun volume bot's custody model is the single most important architectural decision, and the correct answer is always non-custodial. The reference implementation:

- 🔑 **Never asks for a seed phrase or primary private key.**
- 💰 **Deposit wallet is owner-controlled.** The user funds a deposit address with the exact session fee; the engine generates ephemeral sub-wallets from that deposit, runs the campaign, and destroys them at session end.
- ♻️ **Instant refund.** Unused SOL returns to the depositor wallet the moment the session is stopped — no withdrawal queue, no manual clawback.
- 🎯 **Per-transaction key rotation.** Every signature uses an ephemeral keypair; on-chain forensics tools that rely on cluster analysis have no signal to lock onto.
- 🛡️ **Jito private bundle routing.** Every trade is submitted through Jito's private mempool, removing the sandwich and front-running surface that costs naive routing roughly eleven basis points of fill quality per trade.

There is no scenario in which the platform can move funds outside the parameters the depositor signed for.

---

## 💼 Flat 2% Commission

The pricing model of a Solana volume bot is a tell about its business model. Subscriptions and per-feature unlocks exist to maximise revenue, not to align cost with output. A flat percentage commission on session volume ties cost directly to deliverable.

The reference Pump.fun volume bot model:

| What | Detail |
|---|---|
| **Commission** | Flat 2% of target session volume |
| **Session range** | 50 SOL minimum, 5,000 SOL maximum |
| **Coverage** | Solana network fees, priority fees, Jito tips, wallet fleet funding, dust cleanup, auto-comments, auto-favorites, MEV shielding, optional cross-DEX mirroring, 24-hour Telegram support |
| **Hidden costs** | None — no top-ups, no priority-gas surcharges, no per-wallet markups, no setup fees |

Example: a 100 SOL session costs 2 SOL all-in. A 500 SOL session costs 10 SOL all-in. The math is reconcilable in advance, the audit trail is on Solscan, and the cost line items are documented.

---

## 🧠 Wallet Fleet & Anti-Detection

The wallet fleet is the structural foundation of any Solana volume bot. The reference architecture:

- 🆕 **Fresh per session.** Between roughly 100 and 400 wallets per session, drawn from a rotating internal pool. No address reuse across sessions.
- 🎲 **Randomised funding.** No two wallets receive identical SOL amounts at session start.
- 🚫 **Anti-cluster guard.** No two fleet wallets share a Solana block.
- 🌍 **Regional IP routing.** Sub-wallets reach the chain through geo-distributed RPCs across North America, Europe, APAC, and South America.
- ⏱️ **Poisson-distributed timing.** Inter-trade intervals are drawn from a probability distribution, never a fixed schedule.
- ☕ **Natural micro-pauses.** The engine inserts 30-to-120-second idle gaps every 8 to 20 fills — a calibrated simulation of a real trader stepping away from a screen.

The cumulative effect is an on-chain footprint that does not carry the metronome signature anti-detection heuristics on Photon, Trojan, and Bubblemaps-style cluster tools watch for.

---

## 🌍 Multilingual Comment Layer

Pump.fun and Bonk.fun weight comments and watchlist activity alongside trade flow when their trending algorithms sample. A Pump.fun volume bot that ignores the social signal layer is solving half the problem.

The reference implementation:

- 💬 **10,000+ curated comments.** AI-augmented, rotated daily, shuffled per session.
- 🌐 **12 languages, native dialect.** English, Chinese, Korean, Japanese, Turkish, Spanish, Portuguese, French, German, Russian, Vietnamese, Thai — written in regional slang, not machine-translated.
- ⌨️ **Typing-cadence noise.** Per-character timing jitter so messages do not carry the instant-paste signature anti-bot heuristics detect.
- 🎭 **Four wallet personas.** Whale, retail, dev, skeptic — each with a distinct trade-size profile, timing distribution, comment voice, and emoji palette.
- ⚖️ **Sentiment mixer.** Bullish, neutral, and skeptical voices in tunable proportions so the social tape does not collapse into one-note bullishness.
- 🔄 **Auto-reply threads.** Contextual responses to genuine user comments to seed conversation.
- ⭐ **Auto-favorite layer.** Distinct wallets star the token to lift the watchlist-velocity signal.

The native-dialect comment library is the differentiator most operators underestimate. A token whose chat reads as natively local in twelve languages reads as global community activity; a token whose chat is obviously translated English reads as a bot.

---

## ❓ Frequently Asked Questions

### What makes a multi-DEX Pump.fun volume bot different from a single-venue tool?

A single-venue tool — usually one that handles Pump.fun's bonding curve and nothing else — drops the session the moment a token migrates to Raydium. A multi-DEX Solana volume bot tracks the full lifecycle: bonding curve → graduation → AMM depth → aggregator routing → discovery surface. The difference shows up at the migration boundary, which is also the visibility peak of the launch.

### Why does Jupiter consultation matter on every swap?

Jupiter resolves the cheapest multi-hop route across the full Solana DEX landscape. On any given swap, the cheapest route is sometimes a direct pool fill on Raydium, sometimes a two-hop route through Orca, sometimes a three-hop route the operator would never construct by hand. A Solana volume bot that consults Jupiter at every swap captures the best fill automatically.

### How does the bot handle the Pump.fun → Raydium migration?

Block-by-block monitoring of the bonding curve. The moment graduation lands, the routing target switches from the Pump.fun program to the appropriate Raydium pool — CLMM or v4 — without pausing the active session. Optional cross-DEX mirroring across Meteora and Orca runs in parallel during the post-migration phase to distribute emerging depth.

### What is Bonk.fun and why does the bot need separate logic for it?

Bonk.fun is the second credible Solana launchpad, operating under the Bonk ecosystem brand. Its bonding-curve mechanics are similar to Pump.fun but the native chat UI uses a different wire format, the trending algorithm samples differently, and the audience overlap is partial. A Pump.fun volume bot that treats Bonk.fun as a clone of Pump.fun fails on the chat layer and underperforms on the trending algorithm.

### Does the bot move price?

A Pump.fun volume bot produces visibility — trending placement, holder distribution, comment activity, watchlist velocity. It does not produce price discovery. The volume is real on-chain volume, but the deliverable is attention, not price level. A weak token with a strong Solana volume bot session is still a weak token. A strong token with the right session reaches discovery.

### Why timing to DexScreener's `:00 / :15 / :30 / :45` windows?

DexScreener's trending algorithm samples pair activity at minute-edge intervals. A volume spike of equal size lands with disproportionately more visibility when it coincides with a sampling window than when it lands halfway between two windows. A burst-mode Solana volume bot times its spikes to land at the sampling moments.

### How much does a typical session cost?

The flat 2% commission applies to target session volume. A 50 SOL session costs 1 SOL. A 100 SOL session costs 2 SOL. A 500 SOL session costs 10 SOL. The full cost — gas, priority fees, Jito tips, wallet fleet funding, comments, favorites, cross-DEX mirroring — is inside that number.

### What does "non-custodial" actually mean here?

The deposit wallet is owned by the user. The Pump.fun volume bot generates ephemeral session sub-wallets from that deposit, runs the campaign, sweeps the dust at session end, and returns any unused SOL to the depositor. The engine never holds funds outside an active session, never asks for a seed phrase, and never has the authority to move funds outside parameters the depositor signed for.

### Will the bot work if my token is already on Raydium?

Yes. The reference Solana volume bot is multi-DEX from the start: tokens already on Raydium are handled by the AMM routing layer directly, with optional cross-DEX mirroring into Orca and Meteora and Jupiter consultation on every swap.

### Is the audit trail real?

Every fill is an on-chain transaction visible on Solscan. The dashboard streams transaction hashes in real time and the post-session CSV export ties every transaction to the session wallet that produced it. The verification path is a standard Solana explorer lookup; nothing is hidden behind a closed UI.

---

## 🎬 Conclusion

A Solana token launch in the current market is a six-venue event. Pump.fun for the bonding curve. Bonk.fun for the alternative launchpad. Raydium for the migration target. Orca for distributed depth. Jupiter for aggregator routing. DexScreener for discovery. A Pump.fun volume bot that handles any subset of those venues is solving a subset of the problem; the launch outcome is determined by how all six are coordinated.

The reference implementation that handles all six natively — bonding-curve calls on the launchpads, tick-aware execution on the AMMs, per-swap Jupiter consultation, minute-edge timing for the discovery surface, block-by-block migration detection, and a non-custodial wallet fleet underneath everything — is [**https://www.pumpbot.net**](https://www.pumpbot.net/).

The six-venue framework is the floor; configuration against the specific launch in front of the operator is the ceiling. A Solana volume bot that gets the floor right and exposes the ceiling as configurable is the tool a serious launch operator runs. Everything else is theatre.
