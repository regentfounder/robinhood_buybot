# Project-owned buy alerts

## Ownership and access

Create the Telegram bot in the project's BotFather account. The project controls its name,
username, profile photo and description. Use its token as BUYBOT_TOKEN in .env.
Keep the source, server account, RPC subscription, SQLite database and creative assets under
project control. Deployment needs the bot token and read-only RPC URLs. No wallet, signing
key, seed phrase, approval or wallet connection is used by this application.

## Configure alerts

Add the bot to a group, disable BotFather privacy mode so it receives settings replies, and
have an administrator run /add with the token contract address. Open /settings.

- Minimum buy applies before tier selection.
- Standard, Large and Very Large default to USD 0, 500 and 2000.
- Each USD button accepts a decimal threshold. Thresholds must strictly increase.
- A tier starts at its threshold and ends below the next tier's threshold.
- Each tier has its own photo, GIF or MP4 upload and clear control.
- Legacy Media supplies a fallback image when a tier has no upload. Clear it for text-only tiers.
- Pause alerts turns delivery OFF without removing token discovery, tier settings or branding.
- Resume turns delivery ON. Buys processed while paused are intentionally skipped.

Branding / tiers accepts a JSON profile. Only supplied fields change; omitted fields and
existing tier media are retained. Empty social URLs remove buttons. Templates use Telegram
HTML formatting; placeholder values are HTML-escaped automatically.

```json
{
  "website": "https://example.com",
  "x": "https://x.com/yourproject",
  "telegram": "https://t.me/yourproject",
  "template": "<b>{symbol} | {tier} Buy</b>\nSpent: {usd}\nReceived: {tokens} {symbol}\nPrice: {price}\nMarket cap: {marketCap}\nLiquidity: {liquidity}\n<a href=\"{buyerUrl}\">Buyer</a> | <a href=\"{txUrl}\">Transaction</a>",
  "tiers": [
    { "name": "Standard", "minUsd": 25 },
    { "name": "Large", "minUsd": 500 },
    { "name": "Very Large", "minUsd": 2000 }
  ]
}
```

Placeholders: symbol, tier, usd, tokens, buyer, tx, price, marketCap, liquidity, chart,
buyerUrl and txUrl. Templates are limited to 600 source characters. Long expanded captions
are sent as media followed by text. Invalid Telegram HTML falls back to escaped text.
Templates expanding beyond 3500 characters fall back to the default card.
Alerts contain no advertisements or trending promotions, even if legacy ad fields are set.

## Data and reliability

The supported venues are the existing Pons curve and Uniswap V2/V3/V4 decoders on chain 4663.
Discovery currently supports ETH/WETH quote assets. Other quote assets and venues require
additional decoders and valuation support.

USD spent is the decoded quote amount multiplied by the USDG/WETH oracle price at the buy's
block, displayed with two decimals. This is an on-chain USD estimate, not a fiat settlement
receipt. Tokens received retain their raw token decimals. Market cap uses total supply times
the decoded token price; it is not independently verified circulating market cap.
V2 reserve data and V3 pool balances provide liquidity at the transaction block. V4 uses
indexed GeckoTerminal total pool value, cached for one minute. Unindexed pools and curves
show Unavailable rather than fabricated liquidity. Indexer outages do not stop alerts.

WebSocket logs provide the fast path. RPC log scans verify every block before advancing the
persistent cursor, including downtime longer than the old backfill window. Processing
failures and missing USD prices leave the cursor retryable. Persistent SQLite outbox jobs
survive restarts, do not drop during bursts and retry failed sends. Delivery keys suppress
duplicate jobs on log replay. Telegram rate limits can delay delivery under sustained load.

Exactly-once Telegram delivery cannot be guaranteed: a timeout or restart after Telegram
accepts a message but before its acknowledgment is recorded can produce a duplicate.
Live provider coverage, historical calls, supported pools and recovery must be checked with
the project's actual token before production acceptance. Alerts use live events and do not
retract messages on a subsequent chain reorganization.

## Hosting

Use one continuously running Node 20+ process, persistent disk for data and logs, and an
HTTP/WebSocket RPC subscription supporting eth_getLogs, batches and historical eth_call.
Run one instance per bot/database. The HTTP and WebSocket endpoints must be on chain 4663.
No public inbound port is needed: Telegram uses outbound long polling.

Install with npm ci, then run npm run typecheck and npm test. Configure .env from
.env.example on the server and protect it from other users. Run npm start for live alerts.
The dry command still needs RPC access, but it logs cards without sending to Telegram.
Use a separate DB_PATH for dry runs to keep their state separate from production.

deploy/robinhood-buybot.service is a Linux systemd template. Install the project at
/opt/robinhood-buybot, create an unprivileged buybot service user, create writable data and
logs directories owned by that user, install dependencies, and adjust the npm executable
path to the installed Node runtime. Install and enable the service using the server's normal
systemd procedure. Keep tsx installed because npm start executes TypeScript directly.

## Maintenance

Back up SQLite consistently using SQLite's backup command/API, or stop the service for a
filesystem backup including the WAL. Do not copy only the live main database file.
Monitor logs/bot.log and service status for RPC errors, oracle recovery delays, Telegram
send failures and increasing alert_outbox size. Rotate logs and budget disk capacity.
Test restored backups periodically. The alert_deliveries ledger grows with buy volume and
must be retained for replay deduplication; include it in capacity planning.

Monthly costs belong to the project's hosting and RPC accounts and depend on traffic and
provider plans. There is no built-in subscription, ad placement or developer fee in the code.
Maintenance includes dependency updates, RPC quota review, chain/DEX changes and recovery
checks. A commercial development or maintenance quotation is a separate vendor agreement.
