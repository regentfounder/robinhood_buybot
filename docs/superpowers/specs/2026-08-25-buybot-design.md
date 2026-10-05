# Robinhood Chain Telegram Buybot — Design

**Date:** 2026-08-25
**Repo:** https://github.com/blockfile/robinhood-buybot
**Status:** approved in brainstorm; awaiting written-spec review

## 1. Goal

A self-serve Telegram bot that any group admin can add to their token's community group. On every
buy of a tracked token on **Robinhood Chain** (EVM L2, chain ID 4663) it posts a card: emoji row
scaled to buy size, ETH + USD amount, tokens received, tx/buyer links, New Holder / Position +%,
market cap, an operator ad line, and Chart / Buy / 🔥 Get Trending buttons. Groups configure
emoji, $-per-emoji step, minimum buy, and an uploaded GIF/video/photo per token.

Trade sources: **Pons.family V2 bonding curve** and **all Uniswap V2 / V3 / V4 pools** on Robinhood
Chain. This also covers Pons V1 tokens (canonical Uniswap V3 pools) and graduated Pons V2 tokens
(canonical Uniswap V4 pools with the Pons hook). RobinFun is out of scope.

Non-goals for v1: sell alerts, whale tiers, buy competitions, paid ad-free tier, multi-chain,
webhooks, custom layouts. The design leaves room for them (per-group settings table, normalized
`Trade` type) but nothing is built for them.

## 2. Architecture

New repository, same stack as `robinhood-trending`: TypeScript on Node ≥ 20 run with `tsx`,
`better-sqlite3` (WAL), `ethers` v6, `dotenv`, `vitest`, run under pm2 as one long-lived process.
Files `src/telegram.ts`, `src/db/index.ts`, `src/logger.ts`, `src/promo/erc20.ts` and the
test-DI pattern (every network client takes an injected `fetchFn`) are copied from
`robinhood-trending` and adapted; nothing is shared at runtime. The bot has its **own Telegram bot
token**; it never touches the trending bot's chat or token.

```
Telegram getUpdates loop ─► commands/ (/start /add /remove /list /settings, callbacks, my_chat_member)
                                 │ read/write
                                 ▼
                           SQLite (tokens, pools, groups, group_tokens, seen_logs, meta)
                                 ▲ read
ChainWatcher ── WS eth_subscribe(logs) per pool, getLogs backfill, poll fallback
      │ raw log
      ▼
decoders/ (uniV2 | uniV3 | uniV4 | ponsCurve) ─► Trade (normalized) ─► buys only
      │
      ▼
enrich/ (ETH/USD, market cap, holder status) ─► EnrichedBuy
      │
      ▼
poster/ (per-group settings → card → send queue → sendAnimation | sendVideo | sendPhoto | sendMessage)
```

Module boundaries (each is independently testable with fakes):

| Module | Responsibility | Depends on |
|---|---|---|
| `src/chain/rpc.ts` | JSON-RPC over HTTP (`eth_call`, `eth_getLogs`, `eth_blockNumber`, `eth_getTransactionByHash`, `eth_getBalance`), batched, 15 s timeout, injected `fetchFn` | — |
| `src/chain/ws.ts` | WebSocket `eth_subscribe` (`logs`, `newHeads`), reconnect with backoff 1 s→30 s, staleness watchdog, emits raw logs | `ws` |
| `src/chain/watcher.ts` | Owns the subscription set, global block cursor, backfill, dedup, poll fallback; emits raw logs to decoders | rpc, ws, db |
| `src/decoders/*.ts` | Pure functions `decode(log, pool): Trade \| null` for V2, V3, V4, Pons curve | — |
| `src/discovery.ts` | Given a token address: token metadata + all pools across venues | rpc, GeckoTerminal fetch |
| `src/enrich.ts` | ETH/USD price, market cap, holder status | rpc |
| `src/card.ts` | Pure `formatCard(buy, settings, cfg): {text, buttons}` | — |
| `src/poster.ts` | Send queue with global + per-chat rate limits; 403 handling | telegram, db |
| `src/commands/*.ts` | Telegram update handling, admin checks, settings state machine | telegram, db, discovery |
| `src/db/index.ts` | Schema, migrations, typed queries | better-sqlite3 |
| `src/index.ts` | Wiring, startup, shutdown | all |

## 3. Chain facts and addresses (verified 2026-08-25, see research doc)

- Chain ID 4663, gas token ETH, ~100 ms blocks (≈10 blocks/s). 10,000 blocks ≈ 17 min.
- RPC: `RH_RPC_URL` (QuickNode HTTP) and `RH_WS_URL` (QuickNode WSS). `eth_subscribe("logs")`
  works; `eth_getLogs` is capped at a 10,000-block range per call.
- Explorer: `https://robinhoodchain.blockscout.com` (`/tx/<hash>`, `/address/<addr>`).
- WETH `0x0Bd7D308f8E1639FAb988df18A8011f41EAcAD73` (18 dec). USDG
  `0x5fc5360D0400a0Fd4f2af552ADD042D716F1d168` (6 dec).
- ETH/USD oracle: USDG/WETH Uniswap V3 0.01% pool `0x52e65b17fb6e5ba00ed806f37afcd2daa50271ca`,
  token0 = WETH, token1 = USDG. `ethUsd = (sqrtPriceX96 / 2^96)^2 × 1e12`.
- Uniswap V2 factory `0x8bceaa40b9acdfaedf85adf4ff01f5ad6517937f`.
- Uniswap V3 factory `0x1f7d7550b1b028f7571e69a784071f0205fd2efa`, fee tiers 100 / 500 / 3000 / 10000.
- Uniswap V4 PoolManager `0x8366a39cc670b4001a1121b8f6a443a643e40951`, StateView
  `0xf3334192d15450cdd385c8b70e03f9a6bd9e673b`.
- Pons V2 launch factory `0x7eD598BcEf8bd9Edd8C97A195C6d13f40801EC7e`; Pons V2 hook
  `0xE5e702641Ea86F4ae6cC3cDaeD2B886f976Be044`.

Event signatures and topic0:

| Venue | Event | topic0 | Subscribe filter |
|---|---|---|---|
| Uni V2 pair | `Swap(address indexed sender, uint256 amount0In, uint256 amount1In, uint256 amount0Out, uint256 amount1Out, address indexed to)` | `0xd78ad95fa46c994b6551d0da85fc275fe613ce37657fb8d5e3d130840159d822` | `address = pair` |
| Uni V3 pool | `Swap(address indexed sender, address indexed recipient, int256 amount0, int256 amount1, uint160 sqrtPriceX96, uint128 liquidity, int24 tick)` | `0xc42079f94a6350d7e6235f29174924f928cc2ac818eb64fed8004e115fbcca67` | `address = pool` |
| Uni V4 PoolManager | `Swap(bytes32 indexed id, address indexed sender, int128 amount0, int128 amount1, uint160 sqrtPriceX96, uint128 liquidity, int24 tick, uint24 fee)` | `0x40e9cecb9f5f1f1c5b9c97dec2917b7ee92e57ba5563708daca94dd84ad7112f` | `address = PoolManager, topics[1] = PoolId` |
| Pons V2 curve | `CurveBuy(address indexed buyer, address indexed recipient, uint256 quoteIn, uint256 tokensOut, uint256 fee, uint256 tax)` | `0xec36bf571f136799e8dc0b0b8bea4b04d8bd3d43de838aab0d5fc21d4cbfc455` | `address = curve` |
| Pons V2 curve | `CurveCompleted(address recipient, uint256 quoteOut, uint256 tokenOut)` — graduation trigger | computed at build time from the signature | `address = curve` |

Buy classification: V2 — token-side `Out > 0` and quote-side `In > 0`. V3 — token-side `amount`
**negative** (pool-delta convention). V4 — token-side `amount` **positive** (swapper-delta
convention). Pons curve — every `CurveBuy` is a buy. Everything else (sells, zero swaps, LP events)
is ignored.

The V3 and V4 sign conventions are opposite; both are covered by fixture tests using the real logs
captured in the research doc.

## 4. Normalized types

```ts
type PoolKind = 'v2' | 'v3' | 'v4' | 'pons_curve';

interface Pool {
  id: string;            // pair/pool/curve address, or 32-byte PoolId for v4 (lowercase hex)
  token: string;         // tracked token address (lowercase)
  kind: PoolKind;
  tokenIs0: boolean;     // token is token0/currency0 (v2/v3/v4); unused for pons_curve
  quote: string;         // WETH address, or 0x0 for native ETH (v4 / pons)
  fee: number | null;    // v3/v4 fee tier, null otherwise
}

interface Trade {
  token: string;
  pool: Pool;
  buy: boolean;
  quoteIn: bigint;       // wei of ETH/WETH spent (gross, incl. fees)
  tokensOut: bigint;     // raw token units received
  priceQuotePerToken: number; // ETH per whole token, from event (v3/v4 sqrtPrice, pons reserves) or amounts (v2)
  recipient: string;     // event-level recipient/to (may be a router)
  txHash: string;
  logIndex: number;
  blockNumber: number;
}

interface EnrichedBuy extends Trade {
  buyer: string;         // resolved wallet: tx.from
  ethUsd: number;
  usd: number;
  marketCapUsd: number | null;
  holder: { isNew: boolean; positionIncreasePct: number | null } | null;
  ponsProgressPct: number | null;   // only while on curve
}
```

## 5. Chain watcher

- **Subscriptions.** One WS connection. On start and whenever the pool set changes, the watcher
  subscribes `eth_subscribe("logs", {address: [...all v2/v3/pons addresses], topics: [[...topic0s]]})`
  plus one subscription for the V4 PoolManager with `topics: [V4_SWAP, [...poolIds]]`, plus
  `newHeads` for the staleness watchdog. Resubscribing means: unsubscribe old, subscribe new.
- **Cursor.** One global `meta.last_block` = highest block fully processed. Updated after each
  batch of logs is dispatched.
- **Backfill.** On start and after any reconnect: `eth_getLogs` from
  `max(cursor + 1, head − watcher.backfillBlocks)` to `head` in ≤ 10,000-block chunks with the same
  address/topic filters, then arm the live subscription. Logs older than `backfillBlocks`
  (default 10,000) are skipped deliberately — a restart must not flood groups with stale buys.
  On a fresh database (no cursor) the cursor starts at `head` and nothing is backfilled.
  Adding a token only resubscribes; buys that happened before `/add` are never posted.
- **Dedup.** `seen_logs(tx_hash, log_index)` primary key; insert-or-ignore before dispatch; rows
  older than 24 h pruned every 10 min. Live and backfill overlap around reconnect boundaries is
  therefore harmless.
- **Ordering.** Logs are dispatched in `(blockNumber, logIndex)` order within a batch.
- **Staleness watchdog.** No `newHeads` for `watcher.staleSeconds` (default 30) → destroy the socket
  and reconnect (backoff 1 s → 30 s, reset on a successful subscription).
- **Poll fallback.** If the WS has been down for `watcher.pollFallbackAfterSeconds` (default 60), a
  poller runs `eth_getLogs` every 2 s from cursor to head with the same filters through the same
  dispatch path. It stops as soon as the WS resubscribes and backfills.
- ethers v6 `WebSocketProvider` has no reconnect; `src/chain/ws.ts` uses the `ws` package directly
  with raw JSON-RPC (`eth_subscribe` / `eth_unsubscribe`), modelled on the removed
  `robinhood-trending` `src/chain/evm.ts` (`git show 29f12b5^:src/chain/evm.ts`).

## 6. Token discovery (`/add`)

Given an address:

1. `eth_call` `symbol()`, `decimals()`, `totalSupply()` (reuse `erc20.ts` ABI-string decoding;
   reject if `symbol()` reverts and `decimals()` reverts — not an ERC-20).
2. Pons V2: `PonsV2LaunchFactory.getLaunchedToken(token)`; if `exists`, record `curve`, `phase`
   (0 = on curve, 1 = swept, 2 = graduated). Phase 0/1 → add a `pons_curve` pool.
3. Uniswap V2: `factory.getPair(token, WETH)`; non-zero → `v2` pool, `tokenIs0` from `token0()`.
4. Uniswap V3: `factory.getPool(token, WETH, fee)` for the 4 tiers; non-zero → `v3` pool each.
5. Uniswap V4: `GET https://api.geckoterminal.com/api/v2/networks/robinhood/tokens/{token}/pools`;
   entries whose dex id is `uniswap-v4-robinhood` or `pons-v2-dex` yield a PoolId; for each,
   `StateView.getSlot0(poolId)` confirms it exists and the pool's currencies are read from
   GeckoTerminal's `base_token` / `quote_token` to set `tokenIs0` (currencies sort by address; native
   ETH `0x0` is always currency0). Only pools quoted in ETH/WETH are kept.
6. Zero pools found → `/add` fails with "No ETH pool found for this token on Pons or Uniswap".

Result is written to `tokens` and `pools`; the watcher is told to resubscribe.

**Pool refresh.** Every `discovery.refreshMinutes` (default 10) steps 2–5 re-run for every active
token; new pools are added, removed pools are kept (cheap, harmless). When a Pons curve emits
`CurveCompleted`, the token is marked phase 1 and a refresh for that token is scheduled every 30 s
until a V4 pool appears (phase 2), at which point the curve subscription is dropped. This is how a
Pons launch is followed from curve to DEX without admin action.

## 7. Enrichment

Per buy, in parallel, each independently failable:

- **ETH/USD**: `slot0()` of the USDG/WETH pool, cached 30 s. Failure → use last cached value; if
  none, `usd` is null and the card omits USD figures.
- **Market cap**: `totalSupply` (cached from discovery) × `priceQuotePerToken` × `ethUsd`. Pons
  supply is 1e9 whole tokens; curve price = `quoteReserve / tokenReserve` from `getReserves()`
  (cheap and exact) rather than derived from amounts.
- **Pons progress**: `realQuoteReserve() / graduationThreshold()` while phase 0.
- **Buyer**: `eth_getTransactionByHash(txHash).from`. Failure → fall back to the event recipient.
- **Holder status**: `balanceOf(buyer)` at `blockNumber − 1` (archive read on QuickNode). `0` →
  New Holder; else `positionIncreasePct = tokensOut / balanceBefore × 100`. Failure → line omitted.

## 8. Telegram flow

**Bot setup (operator):** create a bot in BotFather, disable privacy mode (`/setprivacy` →
Disable) so it receives `/add` in groups. Token goes in `BUYBOT_TOKEN`.

**Admin check:** a command is accepted only if `getChatAdministrators(chat_id)` lists the sender
(cached 5 min per chat) or the sender is the anonymous-admin sentinel (`GroupAnonymousBot`, id
1087968824). Non-admins get a one-line refusal.

**Commands (groups and supergroups):**

- `/add <CA>` — runs discovery; on success stores a `group_tokens` row with default settings and
  replies:
  `✅ Tracking $SYMBOL · Pons curve (63% to graduation) · Uni V3 1%` (venues listed). On failure
  replies with the reason. Max `limits.maxTokensPerGroup` (default 5) tokens per group.
- `/remove <CA>` — deactivates that group_token.
- `/list` — shows tracked tokens and their settings.
- `/settings` — opens the settings menu (below). If the group tracks several tokens the first
  screen is a token picker.

**Settings menu** — one message with an inline keyboard (`callback_data`), edited in place:

```
⚙️ $SYMBOL settings
Emoji: 🐱   Step: $10   Min buy: $25   Media: GIF set
[🎨 Emoji] [📏 Step] [💵 Min buy]
[🎞 Media] [🗑 Clear media]
[✖ Close]
```

- **Emoji** → "Send the emoji to use" → next message from that admin in that chat is parsed; the
  first extended grapheme cluster is stored. Any standard Unicode emoji is accepted; custom
  (premium) Telegram emoji are rejected with a hint, since bots can only send those with a
  Fragment-purchased username.
- **Step** → buttons `$5 $10 $25 $50 $100` + `Custom` (prompt for a number, 1–100,000).
- **Min buy** → buttons `$0 $10 $25 $50 $100` + `Custom` (0–1,000,000).
- **Media** → "Send a GIF, video or photo now (here or in my DM)". The next `animation`, `video`,
  or `photo` message from that admin — in the group or in a private chat with the bot — is stored
  as `{media_file_id, media_type}` for that group_token. Animated GIFs arrive as Telegram
  `animation` (`.gif` or silent `.mp4`), sent back with `sendAnimation`; `video` → `sendVideo`;
  `photo` (largest size) → `sendPhoto`. Telegram `file_id`s are reusable by the same bot
  indefinitely, so no file is downloaded or stored on disk. If the admin has more than one pending
  media prompt (several groups), the DM upload applies to the most recent prompt.
- **Clear media** → text-only cards.
- Pending prompts live in an in-memory map keyed `(admin_user_id)` with a 5-minute expiry; a
  restart drops them, which is acceptable (the admin just re-opens `/settings`).

**Private chat:** `/start` replies with a short description and an "Add me to your group" button
(`https://t.me/<bot>?startgroup=true`). Media uploads in DM are handled as above; everything else
in DM gets the help text.

**Membership:** on `my_chat_member` where the bot's new status is `left`/`kicked`, the group is
marked inactive and its subscriptions are dropped if no other group tracks those tokens. On
`administrator`/`member` in a new group, a welcome message with `/add` instructions is posted.

## 9. Card

Built by pure `formatCard(buy, settings, cfg)`. HTML parse mode. Sent as the caption of the
group_token's media when set (caption limit 1,024 chars is enforced by truncating the ad line
first, then the venue line), else as a text message with link previews disabled.

```
🟢 <b>$SYMBOL New Buy!</b>
🐱🐱🐱🐱🐱🐱🐱🐱

💵 <b>0.195 ETH</b> ($480.72)
🪙 2.24M SYMBOL
🔗 <a href="…/tx/0x…">Tx</a> | <a href="…/address/0x…">Buyer</a>
👤 New Holder!                        ← or "📈 Position +37%"; omitted if unknown
📊 MC: $571.97K                       ← omitted if unknown
🎯 Pons curve · 63% to graduation     ← only while on curve

📢 Ads: Launch your token on …        ← cfg.adLine, omitted if empty
```

Buttons (one row of three): **📈 Chart** → `https://www.geckoterminal.com/robinhood/pools/<poolId>`
for the pool the buy came from; **⚡ Buy** → `https://www.ponsfamily.com/token/<token>` for Pons
tokens (phase 0–2), else `https://app.uniswap.org/swap?chain=robinhood&outputCurrency=<token>`;
**🔥 Get Trending** → `cfg.trendingUrl` (the trending bot's `/trend` deep link). The Pons token URL
pattern is confirmed against the live site during implementation; if it differs, only
`cfg.buyUrls.pons` changes.

Rules: emoji count = `clamp(floor(usd / stepUsd), 1, limits.maxEmojis)` (default max 40). Buys
with `usd < minBuyUsd` are not posted. Amounts: ETH to 3 significant decimals (`0.195 ETH`), USD
with 2 decimals, token amount abbreviated (`2.24M`), MC abbreviated (`$571.97K`). If `usd` is
unknown the emoji row uses one emoji and USD figures are omitted.

## 10. Poster and rate limits

A single send queue drains at most 25 messages/s globally and 1 message per 3 s per chat
(Telegram's practical group limit is ~20/min). If a chat's queue exceeds 20 pending cards, the
oldest are dropped and a single "⚡ N more buys" line is appended to the next card — a token doing
20+ buys per minute must not stall every other group. On HTTP 429 the `retry_after` is honoured
(existing `post()` logic). On 403 (`bot was kicked` / `chat not found`) the group is deactivated.
On 400 `wrong file identifier` the group_token's media is cleared and the card is resent as text.

## 11. Data model (SQLite, WAL)

```sql
tokens(address TEXT PK, symbol TEXT, decimals INT, total_supply TEXT, pons_curve TEXT NULL,
       pons_phase INT NULL, created_at INT, last_refresh INT)
pools(id TEXT PK, token TEXT REFERENCES tokens, kind TEXT, token_is_0 INT, quote TEXT, fee INT NULL,
      active INT DEFAULT 1)
groups(chat_id INTEGER PK, title TEXT, active INT DEFAULT 1, joined_at INT)
group_tokens(chat_id INTEGER, token TEXT, emoji TEXT, step_usd REAL, min_buy_usd REAL,
             media_file_id TEXT NULL, media_type TEXT NULL, added_by INTEGER, active INT DEFAULT 1,
             created_at INT, PRIMARY KEY (chat_id, token))
seen_logs(tx_hash TEXT, log_index INT, seen_at INT, PRIMARY KEY (tx_hash, log_index))
meta(key TEXT PK, value TEXT)   -- last_block
```

Migrations: `CREATE TABLE IF NOT EXISTS` for the whole schema plus a `schema_version` row in
`meta`; future changes are numbered steps applied in order at startup.

## 12. Configuration

`.env` (loaded with dotenv; missing required keys abort startup with a clear message):

| Key | Required | Notes |
|---|---|---|
| `BUYBOT_TOKEN` | yes | the buybot's own token |
| `RH_RPC_URL` | yes | QuickNode HTTP endpoint |
| `RH_WS_URL` | yes | QuickNode WSS endpoint |
| `DB_PATH` | no | default `data/buybot.sqlite` |

`config.json`:

```json
{
  "adLine": "Launch your token on Robinhood Chain — free trending listing",
  "trendingUrl": "https://t.me/<trending_bot>?start=trend",
  "buyUrls": { "pons": "https://www.ponsfamily.com/token/{token}",
               "uniswap": "https://app.uniswap.org/swap?chain=robinhood&outputCurrency={token}" },
  "defaults": { "emoji": "🟢", "stepUsd": 10, "minBuyUsd": 0 },
  "watcher": { "backfillBlocks": 10000, "staleSeconds": 30, "pollFallbackAfterSeconds": 60 },
  "discovery": { "refreshMinutes": 10 },
  "limits": { "maxEmojis": 40, "maxTokensPerGroup": 5 }
}
```

`npm run dry` runs everything but replaces the Telegram sender with a stdout logger, and does not
start the `getUpdates` loop.

## 13. Error handling summary

- A decoder or enrichment failure affects only that log; it is logged and the stream continues.
- RPC errors during discovery make `/add` fail with a retryable message; nothing is persisted.
- WS problems never lose buys inside the backfill window; older ones are intentionally dropped.
- SQLite is the only state; deleting `data/buybot.sqlite` resets the bot (groups must re-`/add`).
- Logging: one rotating file under `logs/` plus stdout, levels `info`/`warn`/`error`, with the
  tx hash and chat id on every post-related line.

## 14. Testing

Vitest, no network in tests:

- `decoders/*.test.ts` — real captured logs: Meeko `CurveBuy`
  (tx `0x1886757785e18d189f6b02242cad0bf96da3cbc47f406eea2c14e5f717468ac7`), a V3 `Swap` from a
  Pons V1 pool, a V4 `Swap` from a graduated Pons pool (tx `0xf1f60d86…728f` in research), a V2
  `Swap`; sells must return `buy: false`; sign conventions asserted per venue.
- `card.test.ts` — snapshot of the card for a full buy, a text-only buy, unknown-USD buy, caption
  truncation at 1,024 chars.
- `commands/*.test.ts` — fake Telegram (records calls) + `new Db(':memory:')`: admin rejection,
  `/add` happy path with a fake discovery, `/settings` state machine including a DM GIF upload,
  `/remove`, kicked-group deactivation.
- `watcher.test.ts` — fake RPC/WS: backfill chunking at 10,000 blocks, dedup across
  backfill/live overlap, cursor advancement, poll fallback engages and disengages.
- `enrich.test.ts` — ETH/USD from a known `sqrtPriceX96`, MC math, New Holder vs position %.
- `spike/ws-smoke.ts` — manual live check against `RH_WS_URL` (not run in CI).

`npm test`, `npm run typecheck` must pass before merge.

## 15. Rollout

1. Create the bot in BotFather, disable privacy, set commands (`add`, `remove`, `list`,
   `settings`).
2. Fill `.env` (reuse the existing QuickNode endpoints), review `config.json` (ad line, trending
   URL).
3. `npm run dry` for one hour with a test token `/add`ed in a private test group to confirm
   decoding and rate.
4. `pm2 start npm --name robinhood-buybot -- start`.
