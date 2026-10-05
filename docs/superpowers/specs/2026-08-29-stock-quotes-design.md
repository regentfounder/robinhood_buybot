# Tokenized-Stock Quote Support (v1.1) — Design

**Date:** 2026-08-29
**Builds on:** `2026-08-25-buybot-design.md` (v1) as merged in `main` at `d7ace84`.
**Status:** approved in brainstorm.

## 1. Goal

Track Pons.family tokens whose bonding curve — and therefore their graduated Uniswap V4 pool — is
quoted in an ERC-20 other than ETH (tokenized stocks such as AMD `0x86923f96303d656e4aa86d9d42d1e57ad2023fdc`
or SPCX `0x4a0e65a3eccec6dbe60ae065f2e7bb85fae35eea`, or USDG), from the first curve buy through
graduation, exactly as ETH-quoted tokens are tracked today. Motivating case: Ryzen Kitty
`0x50d0d0da00ffd195d2d1d2448617ad039855ad2b`, on-curve, quoted in AMD, currently rejected with
"no ETH-quoted pool found".

Non-goals: Uniswap V2/V3 pairs quoted in a stock (Pons graduates to V4 only); on-chain pricing of
the stock (GeckoTerminal is used; on-chain is a follow-up); any change to sells, media, commands.

## 2. Verified facts (2026-08-29)

- Pons `getLaunchedToken(token).pairToken` is the curve's quote asset; `0x0` for native ETH,
  otherwise an ERC-20. Ryzen Kitty: phase 0, curve `0xac9ae71b760b4194f8103dbf56261001372f79fc`,
  pairToken AMD (18 decimals), threshold 16.6655 AMD, 55.8 % progress.
- `CurveBuy(buyer, recipient, quoteIn, tokensOut, fee, tax)` is identical for ERC-20 quotes;
  `quoteIn` is in the quote's decimals. Captured buy: tx
  `0x526751994ff688677e6f88d24bf122f323b231e28e2f1cb5528fdeffcc075a67`, block 48571614, logIndex 40,
  `quoteIn` = 2919745101889322 (0.00292 AMD), `tokensOut` = 56019163424066370000000 (56,019 tokens).
  `getReserves()` on that curve returned quote 17.81 AMD / token 374.28 M → price 4.7587e-8 AMD/token.
- The graduated V4 pool key is `{currency0, currency1, fee = poolFee, tickSpacing, hooks = PONS_V2_HOOK}`
  with currencies **sorted by address** (native `0x0` always first). Verified: Vlad
  `0x133698a17b7b5c2b981555c56c2b00824f517c1b` / SPCX, fee 0, tickSpacing 200 → PoolId
  `0xaf31df1ebfb8232747db1e0109df074d9e5cd8cccb52ec974e9ddf7c5da25b43` (matches GeckoTerminal).
- GeckoTerminal `GET /networks/robinhood/tokens/{address}` returns `data.attributes.price_usd`
  (AMD → "462.81"). Curve pools list native ETH as `0xeeee…eeee`; V4 pools list it as `0x000…0`.

## 3. Data model

`Pool` gains two fields; `pools` gains two columns via migration **v2**:

```ts
interface Pool {
  …existing…
  /** Quote asset symbol for display ("ETH" for native/WETH). */
  quoteSymbol: string;
  /** Quote asset decimals (18 for native/WETH). */
  quoteDecimals: number;
}
```
```sql
ALTER TABLE pools ADD COLUMN quote_symbol TEXT NOT NULL DEFAULT 'ETH';
ALTER TABLE pools ADD COLUMN quote_decimals INTEGER NOT NULL DEFAULT 18;
```
`Db.migrate()` applies these when `schema_version < 2` and then writes `2`. Existing rows read back
as ETH/18. `upsertPool` writes both columns. Test fixtures set `quoteSymbol: 'ETH', quoteDecimals: 18`
for the existing ETH pools.

`Trade.quoteIn` is redefined as **raw quote units** (wei for ETH); nothing else in `Trade` changes.

## 4. Discovery

`discoverPools(address, decimals)`:

1. Pons: accept the curve for **any** `pairToken`. If `pairToken !== NATIVE`, read the quote's
   `symbol()`/`decimals()` in one `batchSettled` (symbol failure → the address's first 6 chars;
   decimals failure → skip the curve and log — the price maths would be wrong). Curve pool:
   `{ kind: 'pons_curve', quote: pairToken, quoteSymbol, quoteDecimals, tokenIs0: false }`.
2. Derived V4 PoolId: `ponsPoolId(token, quote, fee, tickSpacing)` — sort `[token, quote]` by
   lowercase address (native `0x0` sorts first) → `currency0/currency1`; `tokenIs0 = token < quote`
   when `quote !== NATIVE`, else `false`. Verified with StateView `getSlot0 > 0` as today.
3. GeckoTerminal V4 candidates: keep an entry when the other side is `NATIVE`, `WETH`, **or the
   token's Pons `pairToken`**; `tokenIs0 = other === NATIVE ? false : token < other`;
   `quoteSymbol`/`quoteDecimals` from the Pons quote when `other === pairToken`, else `ETH`/18.
4. V2/V3 unchanged (WETH pairs only).
5. `no_pools` error text becomes "No supported pool found for this token on Pons or Uniswap."

`DiscoveredPools` gains `quoteSymbol: string` so `/add` can say
`✅ Tracking $RYZEN · Pons curve (56% to graduation) · quoted in AMD`.

## 5. Price maths

`execPrice(quoteRaw, tokensRaw, tokenDecimals, quoteDecimals)` and
`sqrtPriceToQuotePerToken(sqrtPriceX96, tokenIs0, tokenDecimals, quoteDecimals)` divide by
`10^quoteDecimals` instead of `1e18`. Every decoder passes `pool.quoteDecimals`. `enrich`'s Pons
reserve price uses the same scaling. Result: `priceQuotePerToken` = quote units per whole token.

## 6. Quote → USD (on-chain primary, GeckoTerminal fallback)

Rationale: GeckoTerminal's public API is capped at 30 requests/minute; the hot path must not depend
on it. Verified 2026-08-29: AMD has Uniswap V3 pools vs USDG (fee 100/3000/10000); SPCX has V2 and
V3 pools vs both WETH and USDG. So a stock's USD price is read on-chain from its own pool.

New `src/quoteOracle.ts`:

```ts
interface QuoteRoute {
  kind: 'v2' | 'v3';
  pool: string;
  quoteIs0: boolean;          // the stock is token0 of that pool
  counter: 'usdg' | 'weth';   // the other side
  counterDecimals: number;    // 6 or 18
}
class QuoteUsdOracle {
  constructor(rpc: Rpc, eth: EthUsdOracle, fetchFn = fetch, opts?: { ttlMs?: number; routeTtlMs?: number })
  /** USD per whole unit of `quote` (given its decimals). NATIVE/WETH → EthUsdOracle; USDG → 1;
   *  otherwise the cached route's price, refreshed every ttlMs (default 30 s); last value kept on
   *  failure; null before any success. */
  get(quote: string, quoteDecimals: number, now: number): Promise<number | null>
}
```

**Route discovery** (once per quote, cached `routeTtlMs` = 1 h, re-run on failure): in one
`batchSettled`, `V3.getPool(quote, USDG, fee)` and `V3.getPool(quote, WETH, fee)` for the 4 tiers
plus `V2.getPair(quote, USDG)` and `V2.getPair(quote, WETH)`. For each existing pool read its depth
in one second batch — V3 `liquidity()`, V2 `getReserves()` — and pick the deepest USDG pool if any
(direct USD), else the deepest WETH pool. If none exists, fall back to GeckoTerminal
`GET /networks/robinhood/tokens/{quote}` → `price_usd` (cached 60 s, at most one request per quote
per minute) and log a warning once per quote.

**Price read** (every `ttlMs`): V3 `slot0()` → `sqrtPriceToQuotePerToken`-style maths with the
stock as the "token" and the counter as the "quote" (`quoteDecimals` = counter decimals); V2
`getReserves()` → `counterReserve / stockReserve` scaled by decimals. USDG counter → price is USD;
WETH counter → multiply by `EthUsdOracle.get(now)` (null if the ETH price is unavailable).

`EnrichDeps.oracle` becomes `quoteOracle: QuoteUsdOracle` (the ETH oracle lives inside it).
`enrich` computes `quoteUsd = await quoteOracle.get(pool.quote, pool.quoteDecimals, now)`,
`usd = quoteIn / 10^quoteDecimals × quoteUsd`, `marketCapUsd = supply × price × quoteUsd`.
`EnrichedBuy` gains `quoteUsd: number | null` (replacing `ethUsd`; no consumer needs ETH specifically).

Thin-pool caveat: the deepest pool is chosen by on-chain depth, so a manipulated dust pool cannot
be selected over a real one; a stock with only dust pools shows a possibly-off USD label, never a
wrong amount.

## 7. Card and logs

`fmtEth(wei)` becomes `fmtQuote(raw, decimals, symbol)` with the same formatting rules
(3 decimals ≥ 1, 3 significant below, trailing zeros trimmed): `0.195 ETH`, `0.0029 AMD`,
`12.5 USDG`. The 💵 line reads `💵 <b>0.0029 AMD</b> ($1.35)`. Emoji row, min-buy, MC, holder,
venue, ad line and buttons are unchanged (all USD-based). The pipeline log line uses `fmtQuote`.

## 8. Refresher / pipeline

No behavioural change beyond the derived PoolId now handling ERC-20 quotes. `PoolIndex`,
`Watcher`, `Poster`, commands: unchanged.

## 9. Error handling

- Gecko down / no price for the quote → `quoteUsd` null → card shows the quote amount without a
  USD figure, one emoji, no MC; min-buy not applied; the buy still posts.
- Quote `decimals()` unreadable at `/add` → curve skipped with a warning; `/add` may still succeed
  via other pools or fail with `no_pools`.
- Migration failure (e.g. read-only DB) → startup error, as today.

## 10. Tests

- `decoders`: AMD-quoted `CurveBuy` fixture (above) → `buy`, `quoteIn 2919745101889322n`,
  `tokensOut 56019163424066370000000n`, price ≈ 5.212e-8 AMD/token (execution price); existing
  fixtures gain `quoteSymbol/quoteDecimals`; `sqrtPriceToQuotePerToken` with `quoteDecimals = 6`.
- `discovery`: Pons tuple with `pairToken = AMD`, phase 0 → curve pool carries AMD/18; phase 2 →
  derived PoolId with sorted currencies (`ponsPoolId` unit test reproduces the Vlad/SPCX id);
  Gecko candidate quoted in the pairToken is kept, in an unrelated token is dropped.
- `db`: a `pools` row inserted under schema v1 (simulate by creating the table without the new
  columns, then constructing `Db`) reads back with `ETH`/18; `schema_version` becomes 2.
- `quoteOracle`: NATIVE/WETH delegate to the ETH oracle; USDG = 1; route discovery picks the
  deepest USDG V3 pool over a WETH pool (fake RPC routes for `getPool`/`getPair`/`liquidity`/
  `getReserves`/`slot0`); a WETH-only route multiplies by ETH/USD; price cached within `ttlMs`
  (second call makes no RPC); last value kept on RPC failure; no pools at all → GeckoTerminal
  fallback with `price_usd` parsed, at most one fetch per minute; null before any success.
  `enrich` on the AMD fixture with a USDG route priced at 462.81 → usd ≈ 1.35, MC uses the AMD
  price.
- `card`: `fmtQuote` cases; a card for an AMD buy shows `💵 <b>0.0029 AMD</b> ($1.35)`.
- `pipeline`: unchanged tests updated for the new deps shape.
- Live: `npm run dry -- --token 0x50d0d0da00ffd195d2d1d2448617ad039855ad2b` posts AMD-quoted cards.

## 11. Follow-ups

On-chain quote pricing (read the stock's deepest WETH/USDG pool); V2/V3 stock pairs; showing the
quote's own USD price on the card.
