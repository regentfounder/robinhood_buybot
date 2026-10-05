# Robinhood Chain Buybot

Self-serve Telegram bot that posts a card in your token's group on every buy on **Robinhood Chain**
(chain 4663) — from the **Pons.family V2 bonding curve** and every **Uniswap V2 / V3 / V4** pool
(which also covers Pons V1 tokens and graduated Pons V2 tokens).

```
🟢 $SYMBOL Standard Buy!
🐱🐱🐱🐱🐱🐱🐱🐱

💵 0.195 ETH ($480.72)
🪙 2.24M SYMBOL
🔗 Tx | Buyer
👤 New Holder!
📊 MC: $571.97K
🎯 Pons curve · 63% to graduation

Price: $0.00057197
Liquidity: $117,420.00
[📈 Chart] [⚡ Buy]
[Website] [X] [Telegram]
```

## How it works

One Node process. A `getUpdates` loop handles `/add`, `/settings`, `/list`, `/remove` and stores
per-group settings in SQLite. A `Watcher` keeps one WebSocket `eth_subscribe("logs")` filtered to
the tracked pools (address filter for V2/V3/Pons, PoolId filter on the V4 PoolManager), backfills
gaps with `eth_getLogs` and falls back to polling if the socket stays down. Pure decoders turn
each log into a normalised trade; the enricher adds ETH/USD (from the USDG/WETH pool), market cap,
the buyer wallet (`tx.from`) and New-Holder / Position +% (`balanceOf` at the previous block); the
rate-limited poster sends the card — with the group's uploaded GIF/video/photo — to every group
tracking that token. A Pons curve that graduates is followed automatically to its V4 pool.

Design: `docs/superpowers/specs/2026-08-25-buybot-design.md`. Research:
`docs/superpowers/research/2026-08-25-buybot-research.md`.

## Setup

1. BotFather: create the bot, **disable privacy mode** (`/setprivacy` → Disable) so it sees `/add`
   in groups. Put the token in `.env` as `BUYBOT_TOKEN`.
2. `cp .env.example .env` and fill in `RH_RPC_URL` / `RH_WS_URL` (QuickNode or Alchemy Robinhood
   Chain endpoints; the public RPC has no WebSocket and is rate-limited).
3. Edit `config.json` defaults, then configure tier thresholds, media and branding with `/settings`.
4. `npm install`, then `npm test` and `npm run typecheck`.

## Running

```
npm run dry -- --token 0xYourToken   # logs cards to stdout for one token, sends nothing
npm start                             # live
pm2 start npm --name robinhood-buybot -- start
```

## Group usage

| Command | Who | What |
|---|---|---|
| `/add 0x…` | admin | discover pools (Pons curve, Uni V2/V3/V4) and start posting |
| `/settings` | admin | tiers, USD thresholds, separate media, branding, links, minimum buy, pause/resume |
| `/list` | admin | tracked tokens and their settings |
| `/remove 0x…` | admin | stop posting for that token |

Media upload: press **🎞 Media** in `/settings`, then send the GIF/video/photo in the group **or in
a DM to the bot**. Only ETH/WETH-quoted pools are tracked in this version.

## Config

| Field | Meaning |
|---|---|
| `adLine` / `trendingUrl` | legacy fields; ignored for white-label alerts |
| `buyUrls.pons` / `.uniswap` | ⚡ Buy button templates (`{token}` placeholder) |
| `defaults.emoji` / `stepUsd` / `minBuyUsd` | per-group defaults applied on `/add` |
| `watcher.backfillBlocks` | legacy field; cursor recovery now scans the complete gap |
| `watcher.staleSeconds` | no WS message for this long → reconnect |
| `watcher.pollFallbackAfterSeconds` | WS down this long → poll `eth_getLogs` every 2 s |
| `discovery.refreshMinutes` | how often pools are re-discovered for tracked tokens |
| `limits.maxEmojis` / `maxTokensPerGroup` | caps |

State lives in `data/buybot.sqlite` (delete it to reset — groups must `/add` again). Logs in
`logs/bot.log`.

See [operations and customization](docs/operations.md) for the branding JSON example,
data accuracy and recovery limits, project ownership and hosting instructions.
