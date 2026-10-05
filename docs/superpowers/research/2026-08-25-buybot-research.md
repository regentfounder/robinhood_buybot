# Buybot research — Robinhood Chain (2026-08-25)

Compiled from four parallel research agents (Robinfun ecosystem, major buybot features, buy-detection
tech, repo reuse survey). Feeds the buybot design brainstorm.

## 1. Robinfun / Robinhood Chain ecosystem

**Robinfun.live** — pump.fun-style bonding-curve launchpad on Robinhood Chain. Whitepaper:
https://robinfun.live/whitepaper. Constant-product curve with virtual reserves; 1B default supply;
70% on curve / 30% LP reserve; starts ~$4k virtual MC; graduates at $9,300 (~5.74 ETH) → ~$44k MC;
LP seeded into **Uniswap V2**, LP burned; 1% fee each side. RobinBridge (https://robinbridge.xyz)
is Robinfun-affiliated (Relay-powered ETH↔RH bridge).

- Current factory `0x4f85392eefcc26775da9d9bec36f07d1b26e434b` (whitepaper "V6", Blockscout-verified
  as `RobinFunFactoryV7`). Events: `TokenCreated`, `Buy(token idx, buyer idx, ethIn, tokensOut,
  newRealEth, newPriceWeiPerToken)`, `Sell`, `ReadyToGraduate`, `Graduated(token idx, pair idx,
  lpEth, lpTokens, treasuryEth, liquidityBurned)`. All curve trades happen on the factory (not
  per-token contracts). Legacy factories V1–V5 listed in whitepaper.
- $ROBINFUN token `0x56a98db16cf501b686c14ba00a5dec02e87083fa`, V2 pair
  `0xE53377eB912D08e1B0160E5Ea0c626CF162870fF`.
- Telegram bots exist: @RobinfunBuyBot, @robinfunbot — **no public docs** on commands/pricing.

**Competing RH-chain buybots (confirmed):**
- bow.fun suite: @bowbuy_bot ("add me to your group, type /add"), @bowtrending_bot (top 10, whale
  alerts, boosts). bow.fun launchpad graduates to Uniswap V3.
- Robinhood Checker @hoodchecker_bot — "first buybot on RH chain": tiered whale alerts, animated
  buy-bars, per-tier media, leaderboards, dev-sell alerts, free.
- Open-source: nirholas/robinhood-chain-alert-bot, nirholas/robinhood-volume-alerts.
- Trading bots supporting RH chain + RobinFun: Maestro, Banana Gun, NockBot, HoodBot.

**Chain facts (verified live):** chain ID 4663, ETH gas, Arbitrum Orbit/Nitro, mainnet 2026-07-01,
**~100 ms blocks** (10 blocks/s). Explorer https://robinhoodchain.blockscout.com (Blockscout v11,
Etherscan-compatible `/api?module=logs&action=getLogs`, `/api/v2/smart-contracts/{addr}`). Public RPC
`https://rpc.mainnet.chain.robinhood.com` is rate-limited and has **no WS**. QuickNode (the existing
RH_RPC_URL/RH_WS_URL): HTTP+WSS, `eth_subscribe("logs")` confirmed working; `eth_getLogs`
10k-block range OK, 20k → HTTP 413. Alchemy also supports the chain with WS + webhooks.

**Contracts:** WETH `0x0Bd7D308f8E1639FAb988df18A8011f41EAcAD73`; USDG
`0x5fc5360D0400a0Fd4f2af552ADD042D716F1d168` (6 dec); Multicall3
`0xcA11bde05977b3631167028862bE2a173976CA11`.
Uniswap V2 factory `0x8bceaa40b9acdfaedf85adf4ff01f5ad6517937f`, Router02
`0x89e5db8b5aa49aa85ac63f691524311aeb649eba`. V3 factory
`0x1f7d7550b1b028f7571e69a784071f0205fd2efa`, SwapRouter02 `0xcaf681a66d020601342297493863e78c959e5cb2`,
UniversalRouter `0x8876789976decbfcbbbe364623c63652db8c0904`. V4 PoolManager
`0x8366a39cc670b4001a1121b8f6a443a643e40951`, StateView `0xf3334192d15450cdd385c8b70e03f9a6bd9e673b`.
USDG/WETH 0.01% V3 pool (ETH/USD oracle) `0x52e65b17fb6e5ba00ed806f37afcd2daa50271ca`.
~35 DEXes on GeckoTerminal (Sushi, Pancake, Curve, Ramses, …); no Camelot.

**Measured swap volume (15 min window):** V3 ~47 swaps/s, V4 ~23/s, V2 ~1.7/s, RobinFun 0. →
Chain-wide topic subscriptions are unworkable; always filter by pool address.

## 2. Feature set of major buybots (Bobby, Safeguard, BuyTech, MajorBuyBot, DEXTools, Codex)

**Setup flow (common):** add bot to group → `/add` (or DM `/start`) → pick chain → paste CA → pick
pair (highest-volume default) → live. Settings via `/settings` in DM; admin-only.

**Per-group settings:** emoji, $-per-emoji step (default ~$100; pump.fun bots ~$10), min buy USD,
media (image/GIF/video, optionally tiered by buy size), circulating-supply override, up to 3 custom
buttons / social links, chart provider, show-price toggle, whale alert (wallet value threshold),
ignore-MEV toggle, buy competitions (biggest buy, pinned, time/spot mode), clone-bot (own token).

**Message layout (Safeguard/Bobby standard):**
```
$SYMBOL New Buy!
[emoji row, count = usd / step]
🟢 0.195 ETH ($480.72)
🔗 Tx | Hash            (buyer addr / tx links)
📂 New Holder!  |  Position +X%
📈 MC: $571.97K
📢 Ads: <ad line>       (free tier)
[Chart] [Buy] [Custom] [🔥 Get Trending]
```
Photo/GIF captions cap at 1,024 chars (text 4,096).

**Monetization:** ad line in every free-tier post (Bobby), paid ad-free/custom-layout tier,
trending slots (`/trend`), boost packages priced in native token (BuyTech: 1 BNB/day ad slot).

**Tech hints:** Bobby v1 = two processes (command listener + chain watcher); per buy: swap event →
match subscriptions → reserves for price/MC → ETH price → buyer balance → post; SQLite WAL.
Safeguard walks inner router/aggregator calls to find more buys. Mobula/Bitquery offer paid
real-time trade streams as an alternative to self-indexing.

**Reference repos:** emmarktech/buybot (Node, ethers v5, Telegraf, Mongo — Uni V2 reserves),
TSRohit99/telegrambot-for-WINR (Arbitrum, scans blocks for V3 events), cybor97/duckbuybot (TS,
SQLite, per-chat settings), debugtitan/buybot and RayaneChCh-dev/buybot-crypto-telegram (Solana).

## 3. Buy detection — recommended approach

**Primary:** WS `eth_subscribe("logs")` on QuickNode filtered by `{address: pool, topics:[SWAP]}`
with reconnect + backfill (`eth_getLogs` cursor, ≤10k blocks/chunk). **Fallback:** `eth_getLogs`
cursor polling every 1–3 s over HTTP RPC (same decode path). Degraded fallback: GeckoTerminal
`/networks/robinhood/pools/{pool}/trades` (10–60 s latency). GMGN has **no per-token trade feed**.

**Event decoding:**
- V2 `Swap(sender idx, amount0In, amount1In, amount0Out, amount1Out, to idx)` topic0
  `0xd78ad95f…`. Buy = token-side `Out > 0` and quote-side `In > 0`. Cache `token0()`.
- V3 `Swap(sender, recipient idx, int256 amount0, int256 amount1, sqrtPriceX96, liquidity, tick)`
  topic0 `0xc42079f9…`. Buy = token-side amount negative. Pools per fee tier
  (100/500/3000/10000).
- V4 PoolManager `Swap(bytes32 id idx, sender idx, int128 amount0, int128 amount1, sqrtPriceX96,
  liquidity, tick, fee)` topic0 `0x40e9cecb…`; filter `topics[1]=PoolId`. May use native ETH
  (`currency0 = 0x0`).
- RobinFun factory `Buy` topic0 `0x75a85e7b…`, filter `topics[1]=token`; on `Graduated` switch to
  V2 pair from `topics[2]`. `buyer` exact; `ethIn` gross of 1% fee.

**Derived data:** ETH/USD from USDG/WETH V3 pool `slot0()` (`(sqrtPriceX96/2^96)^2 * 1e12`),
cache 30–60 s. Price: V2 reserves / V3+V4 sqrtPrice / RobinFun `newPriceWeiPerToken`. MC =
`totalSupply × price × ethUsd`. Buyer = `tx.from` (router/aggregator makes `to`/`sender` a
contract). New holder = `balanceOf(buyer, block-1) == 0` (archive state on QuickNode/Alchemy).
Position +% = `tokensOut / balanceBefore`. Dedupe on `txHash+logIndex` (backfill/live overlap).

ethers v6 `WebSocketProvider` has no auto-reconnect (ethers#4721) — hook raw socket `onclose`,
destroy, backfill, recreate.

## 4. Repo reuse survey (d:\robinhood-trending)

**Reusable as-is / with small edits:**
- `src/telegram.ts` — `Telegram(botToken, chatId, fetchFn)` class; `sendTo(chatId)` already
  exists; 429 retry in `post()`. Needs: parameterize `chatId` on `deleteMessage`/`editCaption`/
  `pinChatMessage`; add `sendAnimation`; widen `InlineButton` with `callback_data`; add per-chat
  send queue (Telegram ~20 msg/min/group).
- `src/promo/erc20.ts` — raw JSON-RPC `eth_call` `symbol()` + ABI-string decode. Add `decimals()`.
- `src/promo/payments.ts:39-51` — batched JSON-RPC skeleton (basis for a `getLogs` poller).
- `src/db/index.ts` — `Db` + `meta` kv (ready-made block cursor store); `new Db(':memory:')` tests.
- `src/promo/orderBot.ts` — in-memory DM state machine + narrow port interfaces pattern.
- Test DI pattern: injected `fetchFn` everywhere.
- Removed WS listener recoverable: `git show 29f12b5^:src/chain/evm.ts` (eth_subscribe logs,
  backoff 1s→30s, 120s staleness watchdog, re-subscribe). Also `src/chain/abi.ts`,
  `constants.ts`.

**Not reusable:** `gmgn.ts` (single trending-only `market/rank` endpoint — arbitrary community
tokens will not be in it; need `/v1/token/info` or on-chain reads for symbol/MC).

**Constraints pushing toward a separate project:**
1. Buybot needs its own bot token (users add it to their groups); `getUpdates` is exclusive per
   token and `OrderBot.run()` already owns the trending bot's poller.
2. Zero data-source overlap with the trending pipeline.
3. Trending loop already skips cycles under load; a per-swap hot loop + sync SQLite writes in the
   same event loop would degrade it.
4. Channel-bound Telegram methods need refactoring regardless.

## 5. Pons.family (added after user redirected scope away from RobinFun)

Dominant launchpad on Robinhood Chain: ~54% of all chain activity, ~10x the next venue (NOXA),
167k+ tokens launched, 2.3k graduated. Docs: https://docs.ponsfamily.com/ (V1),
https://docs.ponsfamily.com/v2 (V2, address + event list). No official API/websocket ("index the
factory and the curves"). Dune: https://dune.com/adam_tehc/pons.

| Gen | Model | Where buys happen | Decoder needed |
|---|---|---|---|
| V1 (`pons-dot-family`) | No curve. Factory mints 1B and seeds a canonical **Uniswap V3** pool (fee 10000, WETH pair). "Graduation" is a label only. | Uniswap V3 pool per token | standard V3 `Swap` |
| V2 curve (`pons-v2`, live 2026-08-03) | pump.fun-style constant-product curve with phantom quote reserve; **one `PonsV2BondingCurve` contract per token**; quote = native ETH (or approved ERC-20, unverified which); 1% fee; graduates at 4.2 ETH real quote (MC ~20.6 ETH). | curve contract (per token) | **`CurveBuy(address indexed buyer, address indexed recipient, uint256 quoteIn, uint256 tokensOut, uint256 fee, uint256 tax)`** topic0 `0xec36bf571f136799e8dc0b0b8bea4b04d8bd3d43de838aab0d5fc21d4cbfc455`; sell `CurveSell` `0x8113d738...`; `CurveCompleted` on the graduating buy |
| V2 Dex (`pons-v2-dex`) | Post-graduation: canonical **Uniswap V4** pool with `V2MemeHook` `0xE5e702641Ea86F4ae6cC3cDaeD2B886f976Be044`; key `{currency0=0x0 (native ETH), currency1=token, fee=0, tickSpacing=200, hooks}`; liquidity permanently locked. | PoolManager singleton, `topics[1]=PoolId` | standard V4 `Swap` (swapper-delta: buy iff token-side amount **positive**) |

**Contracts (Blockscout-verified):** V1 `PonsLaunchFactory` `0xA5aAb3F0c6EeadF30Ef1D3Eb997108E976351feB`;
V2 `PonsV2LaunchFactory` `0x7eD598BcEf8bd9Edd8C97A195C6d13f40801EC7e` (events `TokenLaunched(token idx,
curve idx, deployer idx, pairToken, launchConfigId, graduationThreshold)`, `LaunchSwept`,
`PoolGraduated(token idx, positionId, tokenAmount, pairTokenAmount)`); `PonsV2LaunchDeployer`
`0x3711ceA4feaDE896C913C68F01Eda97Cb06D1A42`; `PonsV2LaunchAndBuy` forwarder
`0xe33E9E479dF8802cb0866d5d05258bEc4cF62948`.

**Discovery:** `PonsV2LaunchFactory.getLaunchedToken(token)` → `(token, curve, deployer, ..., pairToken,
graduationThreshold, poolFee, tickSpacing, ..., phase, ..., exists)`; `phase` 0 = on curve, 1 = swept,
2 = graduated. Curve views: `getReserves()` (quote incl. phantom, token), `realQuoteReserve()`,
`graduationThreshold()`, `graduated()`, `pairToken()`, `isNativeQuote()`. PoolId =
`keccak256(abi.encode(PoolKey))` or from `V2MemeHook.PoolRegistered(poolId idx, memecoin, quoteToken,
creator)` / PoolManager `Initialize`. Check `V2MemeHook.launches(poolId).memecoinIsCurrency0`.
V1: `PonsLaunchFactory.getLaunchedToken(token)` or `UniswapV3Factory.getPool(token, WETH, 10000)`.

**Price/MC:** curve spot = quoteReserve/tokenReserve, MC = price x 1e9 (supply 1e27 wei). V4: from
`sqrtPriceX96`. Buyer: `CurveBuy.recipient` (buyer is often the forwarder); V4 `sender` is the router,
use `tx.from`.

**Sample decoded buy** (Meeko curve `0x0e20804b96Cd46528F8915183aF58d0516875306`, tx
`0x1886757785e18d189f6b02242cad0bf96da3cbc47f406eea2c14e5f717468ac7`): quoteIn 0.0099 ETH, tokensOut
2,237,100.69, fee 1%, tax 0.

**Scope decision (user, 2026-08-25):** drop RobinFun. Sources = Pons V2 curve + all Uniswap V2/V3/V4
pools on Robinhood Chain (which also covers Pons V1 and Pons V2 Dex).
