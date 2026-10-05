# Tokenized-Stock Quote Support (v1.1) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Track Pons.family tokens whose curve and graduated V4 pool are quoted in an ERC-20 (AMD, SPCX, USDG, …) end-to-end, pricing the quote in USD from its own on-chain Uniswap pool.

**Architecture:** `Pool` learns its quote's symbol/decimals (schema v2); the price helpers divide by the quote's decimals instead of `1e18`; a new `QuoteUsdOracle` resolves USD per quote unit (ETH/WETH → existing oracle, USDG → 1, other ERC-20 → deepest USDG/WETH Uniswap V2/V3 pool read every 30 s, GeckoTerminal only when no pool exists); discovery accepts any Pons `pairToken`, derives the graduated PoolId with sorted currencies, and admits GeckoTerminal V4 pools quoted in that asset; the card prints `0.00292 AMD ($1.35)`.

**Tech Stack:** unchanged — TypeScript ESM, `better-sqlite3`, `ethers` v6 (`AbiCoder`, `keccak256`), `vitest`. Spec: `docs/superpowers/specs/2026-08-29-stock-quotes-design.md` (read it once; it carries the verified on-chain facts).

## Global Constraints

- Repo `d:\robinhood-buybot`, work on branch `feat/v1.1-stock-quotes` off `main` (`5b3fe09`). `npx vitest run` (currently 112 tests / 16 files) and `npx tsc --noEmit` must pass at the end of every task; never weaken an existing assertion — update it only where this plan says the behaviour changes.
- All addresses lowercase. Constants already exist in `src/chain/constants.ts`: `NATIVE`, `WETH` `0x0bd7d308f8e1639fab988df18a8011f41eacad73`, `USDG` `0x5fc5360d0400a0fd4f2af552add042d716f1d168` (6 dec), `UNI_V2_FACTORY`, `UNI_V3_FACTORY`, `UNI_V3_FEES` (100/500/3000/10000), `UNI_V4_STATE_VIEW`, `PONS_V2_FACTORY`, `PONS_V2_HOOK`, `PONS_LAUNCH_TUPLE`, `GECKO_API`, `SEL.{getPair,getPool,slot0,getReserves,symbol,decimals,…}`.
- Verified facts to reproduce exactly: AMD `0x86923f96303d656e4aa86d9d42d1e57ad2023fdc` (18 dec); Ryzen Kitty token `0x50d0d0da00ffd195d2d1d2448617ad039855ad2b`, curve `0xac9ae71b760b4194f8103dbf56261001372f79fc`; captured AMD `CurveBuy` → `quoteIn 2919745101889322n`, `tokensOut 56019163424066363244767n`, execution price ≈ 5.2120e-8 AMD/token; Vlad `0x133698a17b7b5c2b981555c56c2b00824f517c1b` / SPCX `0x4a0e65a3eccec6dbe60ae065f2e7bb85fae35eea`, fee 0, tickSpacing 200 → PoolId `0xaf31df1ebfb8232747db1e0109df074d9e5cd8cccb52ec974e9ddf7c5da25b43`; TA `0x9ca1cc0c90d97b4f36c5e2232d4fbd705a73c65d` / native, fee 0, tickSpacing 200 → `0x8052650014fbe824973a674946e08182a6a116cd9b72b50a97444bbe8a2d8b86`.
- GeckoTerminal public API is limited to 30 requests/minute: it must never be in the per-buy path; at most one price request per quote per minute, only when no on-chain pool exists.
- Every timestamp-taking function takes `now`. Tests never touch the network.
- Commit after every task with a conventional message ending in `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>`.

## File Structure

| File | Change |
|---|---|
| `src/decoders/types.ts` | `Pool.quoteSymbol`, `Pool.quoteDecimals`; `Trade.quoteIn` doc |
| `src/db/index.ts` | schema v2 (two `pools` columns), migration, `upsertPool`/`mapPool`/`POOL_COLS` |
| `src/decoders/common.ts` + the four decoders | price helpers take `quoteDecimals` |
| `src/chain/constants.ts` | `SEL.liquidity` |
| `src/quoteOracle.ts` (new) | `QuoteUsdOracle`, `QuoteRoute` |
| `src/enrich.ts` | `quoteUsd` replaces `ethUsd`; `EnrichDeps.quoteOracle` |
| `src/card.ts` | `fmtQuote` replaces `fmtEth` |
| `src/pipeline.ts`, `src/index.ts` | wiring |
| `src/discovery.ts` | any `pairToken`; `ponsPoolId(token, quote, fee, tickSpacing)`; quote metadata; `DiscoveredPools.quoteSymbol` |
| `src/commands/texts.ts`, `src/commands/handler.ts` | `/add` reply names the quote; `no_pools` wording |
| `README.md` | stock-quoted tokens + price source |
| `tests/fixtures/logs.ts` | new fields on every pool; `AMD`, `PONS_POOL_AMD`, `PONS_BUY_AMD` |
| `tests/db.test.ts`, `tests/decoders.test.ts`, `tests/quoteOracle.test.ts` (new), `tests/enrich.test.ts`, `tests/card.test.ts`, `tests/pipeline.test.ts`, `tests/discovery.test.ts`, `tests/commands.test.ts`, `tests/abi.test.ts` | per task |

---

### Task 1: Pool quote fields and schema v2 migration

**Files:**
- Modify: `src/decoders/types.ts`, `src/db/index.ts`, `src/discovery.ts` (pool literals only), `tests/fixtures/logs.ts`, `tests/db.test.ts`, `tests/discovery.test.ts`, `tests/commands.test.ts`
- Test: `tests/db.test.ts`

**Interfaces:**
- Produces: `Pool` gains `quoteSymbol: string` and `quoteDecimals: number`; `Db` schema version 2 with `pools.quote_symbol TEXT NOT NULL DEFAULT 'ETH'` and `pools.quote_decimals INTEGER NOT NULL DEFAULT 18`; `upsertPool`/`poolsForToken`/`activePools` round-trip the new fields.

- [ ] **Step 1: Extend the `Pool` type**

In `src/decoders/types.ts`, add two fields to `Pool` after `decimals`, and fix the `Trade.quoteIn` comment:
```ts
  /** Token decimals (needed for price maths). */
  decimals: number;
  /** Quote asset symbol for display ("ETH" for native ETH / WETH). */
  quoteSymbol: string;
  /** Quote asset decimals (18 for native ETH / WETH). */
  quoteDecimals: number;
```
```ts
  /** Buy: raw quote units spent (wei for ETH, gross of fees). Sell: raw quote units received. */
  quoteIn: bigint;
```

- [ ] **Step 2: Update every `Pool` literal so the project typechecks**

Add `quoteSymbol: 'ETH', quoteDecimals: 18` (before the closing `}`) to each of these literals — they are all ETH/WETH-quoted:
- `tests/fixtures/logs.ts`: `V2_POOL`, `V3_POOL_USDG`, `V3_POOL_PUH`, `V4_POOL_TA`, `PONS_POOL` (5 literals).
- `tests/db.test.ts`: `POOL`.
- `tests/discovery.test.ts`: the `toEqual([{ id: CURVE, … decimals: 18 }])` expectation, the `toEqual([{ id: POOLID, … decimals: 18 }])` expectation, and `const base = { token: TOKEN, tokenIs0: false, quote: NATIVE, decimals: 18 }` in the `venueLabel` test.
- `tests/commands.test.ts`: the `pools: [{ id: CURVE, … decimals: 18 }]` literal in `DISCOVERED`.
- `src/discovery.ts`: every object literal that builds a pool — `ponsV4Candidate = { … fee }`, the `pools.push({ id: ponsCurve, … decimals })` curve literal, the V2/V3 `pools.push({ id: c.id, … decimals })`, and `out.push({ id, token, kind: 'v4', … fee })` in `v4Candidates` — gets `quoteSymbol: 'ETH', quoteDecimals: 18` (Task 5 makes these real).

Run: `npx tsc --noEmit` — must be clean before continuing.

- [ ] **Step 3: Write the failing migration test**

Append to `tests/db.test.ts` (add the imports at the top: `import Database from 'better-sqlite3'; import { mkdtempSync, rmSync } from 'node:fs'; import { join } from 'node:path'; import { tmpdir } from 'node:os';`):
```ts
  it('migrates a v1 database: pools gain quote columns with ETH/18 defaults', () => {
    const dir = mkdtempSync(join(tmpdir(), 'bb-mig-'));
    const path = join(dir, 'v1.sqlite');
    const raw = new Database(path);
    raw.exec(`CREATE TABLE pools (id TEXT PRIMARY KEY, token TEXT NOT NULL, kind TEXT NOT NULL, token_is_0 INTEGER NOT NULL,
        quote TEXT NOT NULL, fee INTEGER, decimals INTEGER NOT NULL, active INTEGER NOT NULL DEFAULT 1);
      CREATE TABLE meta (key TEXT PRIMARY KEY, value TEXT NOT NULL);
      INSERT INTO meta VALUES ('schema_version', '1');
      INSERT INTO pools VALUES ('0xpool', '${TOKEN}', 'v2', 0, '0x0bd7d308f8e1639fab988df18a8011f41eacad73', NULL, 18, 1);`);
    raw.close();
    const db2 = new Db(path);
    try {
      expect(db2.getMeta('schema_version')).toBe('2');
      expect(db2.poolsForToken(TOKEN)).toEqual([{ id: '0xpool', token: TOKEN, kind: 'v2', tokenIs0: false,
        quote: '0x0bd7d308f8e1639fab988df18a8011f41eacad73', fee: null, decimals: 18, quoteSymbol: 'ETH', quoteDecimals: 18 }]);
      db2.upsertPool({ ...POOL, id: '0xamd', quote: '0x86923f96303d656e4aa86d9d42d1e57ad2023fdc', quoteSymbol: 'AMD', quoteDecimals: 18 });
      expect(db2.poolsForToken(TOKEN).find((p) => p.id === '0xamd')).toMatchObject({ quoteSymbol: 'AMD', quoteDecimals: 18 });
    } finally {
      db2.close();
      rmSync(dir, { recursive: true, force: true });
    }
  });
```

- [ ] **Step 4: Run the test to verify it fails**

Run: `npx vitest run tests/db.test.ts`
Expected: the new test FAILS (`schema_version` is `'1'` / `no such column: quote_symbol`); the existing 6 pass.

- [ ] **Step 5: Implement schema v2**

In `src/db/index.ts`:
1. `const SCHEMA_VERSION = 2;`
2. In `SCHEMA`, inside `CREATE TABLE IF NOT EXISTS pools (…)`, add after `fee INTEGER,`:
```sql
  decimals INTEGER NOT NULL,
  quote_symbol TEXT NOT NULL DEFAULT 'ETH',
  quote_decimals INTEGER NOT NULL DEFAULT 18,
```
(`decimals` already exists — keep one copy; the two new lines go right after it.)
3. Replace `migrate()`:
```ts
  /** Numbered migrations applied in order; v1 is the base schema, v2 adds pools.quote_symbol/quote_decimals. */
  private migrate(): void {
    const current = Number(this.getMeta('schema_version') ?? '0');
    if (current < 2) {
      const cols = new Set((this.db.prepare(`PRAGMA table_info(pools)`).all() as Array<{ name: string }>).map((c) => c.name));
      if (!cols.has('quote_symbol')) this.db.exec(`ALTER TABLE pools ADD COLUMN quote_symbol TEXT NOT NULL DEFAULT 'ETH'`);
      if (!cols.has('quote_decimals')) this.db.exec(`ALTER TABLE pools ADD COLUMN quote_decimals INTEGER NOT NULL DEFAULT 18`);
    }
    if (current < SCHEMA_VERSION) this.setMeta('schema_version', String(SCHEMA_VERSION));
  }
```
4. `POOL_COLS`:
```ts
const POOL_COLS = `id, token, kind, token_is_0 AS tokenIs0, quote, fee, decimals, quote_symbol AS quoteSymbol, quote_decimals AS quoteDecimals`;
```
5. `mapPool`:
```ts
const mapPool = (r: RawPool): Pool => ({ id: r.id, token: r.token, kind: r.kind as PoolKind, tokenIs0: !!r.tokenIs0, quote: r.quote, fee: r.fee, decimals: r.decimals, quoteSymbol: r.quoteSymbol, quoteDecimals: r.quoteDecimals });
```
6. `upsertPool`:
```ts
  upsertPool(p: Pool): void {
    this.db.prepare(`INSERT INTO pools (id, token, kind, token_is_0, quote, fee, decimals, quote_symbol, quote_decimals, active)
      VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, 1)
      ON CONFLICT(id) DO UPDATE SET token = excluded.token, kind = excluded.kind, token_is_0 = excluded.token_is_0,
        quote = excluded.quote, fee = excluded.fee, decimals = excluded.decimals,
        quote_symbol = excluded.quote_symbol, quote_decimals = excluded.quote_decimals, active = 1`)
      .run(p.id, p.token, p.kind, p.tokenIs0 ? 1 : 0, p.quote, p.fee, p.decimals, p.quoteSymbol, p.quoteDecimals);
  }
```

- [ ] **Step 6: Run the tests and typecheck**

Run: `npx vitest run && npx tsc --noEmit`
Expected: 113 tests PASS (112 + 1); typecheck clean.

- [ ] **Step 7: Commit**

```bash
git add src/decoders/types.ts src/db/index.ts src/discovery.ts tests/fixtures/logs.ts tests/db.test.ts tests/discovery.test.ts tests/commands.test.ts
git commit -m "feat: pool quote symbol/decimals with schema v2 migration

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 2: Price maths honour quote decimals; AMD-quoted curve fixture

**Files:**
- Modify: `src/decoders/common.ts`, `src/decoders/uniV2.ts`, `src/decoders/uniV3.ts`, `src/decoders/uniV4.ts`, `src/decoders/ponsCurve.ts`, `src/enrich.ts` (one line), `tests/fixtures/logs.ts`
- Test: `tests/decoders.test.ts`

**Interfaces:**
- Produces: `execPrice(quoteRaw: bigint, tokensRaw: bigint, tokenDecimals: number, quoteDecimals: number): number`; `sqrtPriceToQuotePerToken(sqrtPriceX96: bigint, tokenIs0: boolean, tokenDecimals: number, quoteDecimals: number): number` (both return quote units per whole token); fixtures `AMD`, `PONS_POOL_AMD`, `PONS_BUY_AMD`.

- [ ] **Step 1: Add the AMD fixture**

Append to `tests/fixtures/logs.ts`:
```ts
// --- Pons V2 bonding curve quoted in AMD (tokenized stock, 18 dec): "Ryzen Kitty" curve, captured 2026-08-29 ---
export const AMD = '0x86923f96303d656e4aa86d9d42d1e57ad2023fdc';
export const PONS_POOL_AMD: Pool = {
  id: '0xac9ae71b760b4194f8103dbf56261001372f79fc', token: '0x50d0d0da00ffd195d2d1d2448617ad039855ad2b',
  kind: 'pons_curve', tokenIs0: false, quote: AMD, fee: null, decimals: 18, quoteSymbol: 'AMD', quoteDecimals: 18,
};
export const PONS_BUY_AMD: RawLog = {
  address: PONS_POOL_AMD.id, blockNumber: 48571614, logIndex: 40, removed: false,
  transactionHash: '0x526751994ff688677e6f88d24bf122f323b231e28e2f1cb5528fdeffcc075a67',
  topics: [TOPIC_PONS_BUY, '0x000000000000000000000000b3e204090f631b47d55b222aeb71e714d9593992', '0x000000000000000000000000b3e204090f631b47d55b222aeb71e714d9593992'],
  data: '0x000000000000000000000000000000000000000000000000000a5f7e21a1ff2a000000000000000000000000000000000000000000000bdcce102bf0aec0c0df00000000000000000000000000000000000000000000000000001a8e0fb2428d00000000000000000000000000000000000000000000000000001a8e0fb2428d',
};
```

- [ ] **Step 2: Write the failing tests**

In `tests/decoders.test.ts`, inside `describe('decodeTrade')` add:
```ts
  it('Pons CurveBuy quoted in a tokenized stock (AMD)', () => {
    const b = decodeTrade(F.PONS_BUY_AMD, F.PONS_POOL_AMD)!;
    expect(b.buy).toBe(true);
    expect(b.quoteIn).toBe(2919745101889322n);
    expect(b.tokensOut).toBe(56019163424066363244767n);
    expect(b.recipient).toBe('0xb3e204090f631b47d55b222aeb71e714d9593992');
    expect(rel(b.priceQuotePerToken, 5.2120e-8)).toBeLessThan(1e-3);
  });
```
Replace the whole `describe('price maths')` block with:
```ts
describe('price maths', () => {
  it('execPrice scales by token and quote decimals and guards zero', () => {
    expect(execPrice(1000000000000000000n, 2000000000000000000n, 18, 18)).toBeCloseTo(0.5, 12);
    expect(execPrice(1000000000000000000n, 2000000n, 6, 18)).toBeCloseTo(0.5, 12);
    expect(execPrice(500000n, 2000000000000000000n, 18, 6)).toBeCloseTo(0.25, 12); // 0.5 USDG for 2 tokens
    expect(execPrice(1n, 0n, 18, 18)).toBe(0);
    expect(execPrice(2919745101889322n, 56019163424066363244767n, 18, 18)).toBeCloseTo(5.2120e-8, 11);
  });
  it('sqrtPriceToQuotePerToken for token0/token1 positions and a 6-decimal quote', () => {
    expect(rel(sqrtPriceToQuotePerToken(3945020819825199951804666n, false, 6, 18), 0.00040333)).toBeLessThan(1e-3);
    expect(rel(sqrtPriceToQuotePerToken(2973922873297686371811199n, true, 18, 18), 1.40897e-9)).toBeLessThan(1e-3);
    // USDG per WETH from the USDG/WETH pool: WETH is token0 (18 dec), USDG is the quote (6 dec)
    expect(rel(sqrtPriceToQuotePerToken(3942406684091758005651998n, true, 18, 6), 2476.07)).toBeLessThan(1e-4);
  });
});
```

- [ ] **Step 3: Run the tests to verify they fail**

Run: `npx vitest run tests/decoders.test.ts`
Expected: FAIL — `execPrice`/`sqrtPriceToQuotePerToken` called with 4 arguments (typecheck error under vitest, or wrong values).

- [ ] **Step 4: Implement**

`src/decoders/common.ts` — replace the two helpers:
```ts
/** Execution price: quote units per whole token. 0 when no tokens moved. */
export function execPrice(quoteRaw: bigint, tokensRaw: bigint, tokenDecimals: number, quoteDecimals: number): number {
  if (tokensRaw === 0n) return 0;
  return (Number(quoteRaw) / 10 ** quoteDecimals) / (Number(tokensRaw) / 10 ** tokenDecimals);
}

/** Uniswap V3/V4 sqrtPriceX96 → quote units per whole token, for the token in position 0 or 1. */
export function sqrtPriceToQuotePerToken(sqrtPriceX96: bigint, tokenIs0: boolean, tokenDecimals: number, quoteDecimals: number): number {
  const ratio = (Number(sqrtPriceX96) / 2 ** 96) ** 2; // raw token1 per raw token0
  const rawQuotePerToken = tokenIs0 ? ratio : 1 / ratio;
  return rawQuotePerToken * 10 ** tokenDecimals / 10 ** quoteDecimals;
}
```
Decoders — every call site passes the pool's quote decimals:
- `uniV2.ts`: `execPrice(quote, tokens, pool.decimals, pool.quoteDecimals)`
- `uniV3.ts` and `uniV4.ts`: `sqrtPriceToQuotePerToken(sqrt, pool.tokenIs0, pool.decimals, pool.quoteDecimals)`
- `ponsCurve.ts` (both branches): `execPrice(quote, tokens, pool.decimals, pool.quoteDecimals)`
- `src/enrich.ts`, the Pons reserve price line: `if (t > 0n) price = (Number(q) / Number(t)) * 10 ** token.decimals / 10 ** trade.pool.quoteDecimals;`

- [ ] **Step 5: Run the tests and typecheck**

Run: `npx vitest run && npx tsc --noEmit`
Expected: 114 tests PASS; typecheck clean.

- [ ] **Step 6: Commit**

```bash
git add src/decoders src/enrich.ts tests/fixtures/logs.ts tests/decoders.test.ts
git commit -m "feat: price maths honour quote decimals; AMD-quoted curve fixture

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 3: `QuoteUsdOracle` — on-chain quote→USD with GeckoTerminal fallback

**Files:**
- Create: `src/quoteOracle.ts`
- Modify: `src/chain/constants.ts` (`SEL.liquidity`), `tests/abi.test.ts` (one assertion)
- Test: `tests/quoteOracle.test.ts`

**Interfaces:**
- Consumes: `Rpc.call/batchSettled` (`src/chain/rpc.ts`), `EthUsdOracle.get(now)` (`src/enrich.ts`), `sqrtPriceToQuotePerToken` (Task 2), constants.
- Produces:
```ts
interface QuoteRoute { kind: 'v2' | 'v3'; pool: string; quoteIs0: boolean; counter: 'usdg' | 'weth'; counterDecimals: number }
interface QuoteOracleOpts { ttlMs?: number; routeTtlMs?: number; geckoTtlMs?: number }   // defaults 30 s, 1 h, 60 s
class QuoteUsdOracle {
  constructor(rpc: Pick<Rpc,'call'|'batchSettled'>, eth: Pick<EthUsdOracle,'get'>, fetchFn?: typeof fetch, opts?: QuoteOracleOpts, logFn?: LogFn)
  get(quote: string, quoteDecimals: number, now: number): Promise<number | null>
  route(quote: string, now: number): Promise<QuoteRoute | null>     // public for tests
}
```

Route ranking: USDG pools beat WETH pools; V3 beats V2; within the same rank the deepest pool wins (V3 `liquidity()`, V2 counter-side reserve). USDG → price is USD directly; WETH → multiply by the ETH oracle. Uniswap sorts pool tokens by address, so `quoteIs0 = quote < counter` needs no `token0()` call.

- [ ] **Step 1: Add the `liquidity()` selector**

In `src/chain/constants.ts` `SEL`, add `liquidity: sel('liquidity()'),` after `getReserves`. In `tests/abi.test.ts` "has the verified selectors and topics", add `expect(SEL.liquidity).toBe('0x1a686502');`.

- [ ] **Step 2: Write the failing test**

`tests/quoteOracle.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { AbiCoder } from 'ethers';
import { QuoteUsdOracle } from '../src/quoteOracle';
import { Rpc } from '../src/chain/rpc';
import { SEL, WETH, NATIVE, USDG, UNI_V2_FACTORY, UNI_V3_FACTORY } from '../src/chain/constants';
import { noopLog } from '../src/logger';

const coder = AbiCoder.defaultAbiCoder();
const AMD = '0x86923f96303d656e4aa86d9d42d1e57ad2023fdc';
const P_USDG_100 = '0x4ee16fe614f3a04f1510edf1932a41cbd5c4af9c';
const P_USDG_3000 = '0x48d284a2a4d3dc1b3da08231fe44317e7e7aa51f';
const P_WETH_V2 = '0x5d5a3f7232cef9788ef231712303ec19db2d5573';
const ZERO = '0x' + '0'.repeat(64);
const addr = (a: string) => coder.encode(['address'], [a]);
const num = (n: bigint | number) => coder.encode(['uint256'], [n]);
// USDG (token0, 6 dec) / AMD (token1, 18 dec) at 462.81 USDG per AMD
const SLOT0_AMD = coder.encode(['uint160', 'int24', 'uint16', 'uint16', 'uint16', 'uint8', 'bool'], [3682800696557432101561618158059520n, 0, 0, 0, 0, 0, true]);
const rel = (a: number, b: number) => Math.abs(a - b) / b;

/** Fake RPC: eth_call routed by "<to>:<selector>" → fn(calldata) → hex; unknown → revert. Counts every eth_call. */
function rpcWith(routes: Record<string, (data: string) => string>) {
  let calls = 0;
  const fetchFn = (async (_u: unknown, init?: RequestInit) => {
    const reqs = JSON.parse(String(init?.body)) as Array<{ id: number; method: string; params: any[] }>;
    const out = reqs.map((r) => {
      if (r.method !== 'eth_call') return { id: r.id, result: '0x1' };
      calls++;
      const key = `${r.params[0].to.toLowerCase()}:${r.params[0].data.slice(0, 10)}`;
      const fn = routes[key];
      return fn ? { id: r.id, result: fn(r.params[0].data) } : { id: r.id, error: { code: 3, message: 'revert' } };
    });
    return new Response(JSON.stringify(out), { status: 200 });
  }) as typeof fetch;
  return { rpc: new Rpc('http://x', fetchFn), calls: () => calls };
}
const has = (data: string, a: string) => data.includes(a.slice(2).toLowerCase());
const feeIs = (data: string, fee: number) => data.endsWith(fee.toString(16).padStart(64, '0'));

function gecko(price: string | null, status = 200) {
  let n = 0;
  const fetchFn = (async () => { n++; return new Response(JSON.stringify(price === null ? {} : { data: { attributes: { price_usd: price } } }), { status }); }) as typeof fetch;
  return { fetchFn, count: () => n };
}
const eth = (v: number | null) => ({ get: async () => v });

describe('QuoteUsdOracle', () => {
  it('ETH/WETH delegate to the ETH oracle; USDG is $1; no RPC involved', async () => {
    const { rpc, calls } = rpcWith({});
    const o = new QuoteUsdOracle(rpc, eth(2476.07), gecko(null).fetchFn, {}, noopLog);
    expect(await o.get(NATIVE, 18, 0)).toBe(2476.07);
    expect(await o.get(WETH, 18, 0)).toBe(2476.07);
    expect(await o.get(USDG, 6, 0)).toBe(1);
    expect(calls()).toBe(0);
  });

  it('picks the deepest USDG V3 pool over a WETH V2 pool and prices from slot0; caches within ttl', async () => {
    const { rpc, calls } = rpcWith({
      [`${UNI_V3_FACTORY}:${SEL.getPool}`]: (d) => (has(d, USDG) && feeIs(d, 100) ? addr(P_USDG_100) : has(d, USDG) && feeIs(d, 3000) ? addr(P_USDG_3000) : ZERO),
      [`${UNI_V2_FACTORY}:${SEL.getPair}`]: (d) => (has(d, WETH) ? addr(P_WETH_V2) : ZERO),
      [`${P_USDG_100}:${SEL.liquidity}`]: () => num(5n * 10n ** 18n),
      [`${P_USDG_3000}:${SEL.liquidity}`]: () => num(9n * 10n ** 18n),
      [`${P_WETH_V2}:${SEL.getReserves}`]: () => coder.encode(['uint112', 'uint112', 'uint32'], [10n ** 24n, 10n ** 24n, 0]),
      [`${P_USDG_3000}:${SEL.slot0}`]: () => SLOT0_AMD,
    });
    const g = gecko('1');
    const o = new QuoteUsdOracle(rpc, eth(2476.07), g.fetchFn, { ttlMs: 30_000 }, noopLog);
    expect(await o.route(AMD, 0)).toEqual({ kind: 'v3', pool: P_USDG_3000, quoteIs0: false, counter: 'usdg', counterDecimals: 6 });
    expect(rel((await o.get(AMD, 18, 0))!, 462.81)).toBeLessThan(1e-6);
    const before = calls();
    expect(rel((await o.get(AMD, 18, 10_000))!, 462.81)).toBeLessThan(1e-6);
    expect(calls()).toBe(before);
    expect(g.count()).toBe(0);
  });

  it('WETH-only route multiplies by the ETH price; null when the ETH price is unknown', async () => {
    const X = '0x0aaa000000000000000000000000000000000001'; // sorts before WETH → X is token0
    const P = '0x1111111111111111111111111111111111111111';
    // 0.2 WETH per X: raw1/raw0 = 0.2 (both 18 dec) → sqrt = sqrt(0.2) × 2^96
    const sqrt = BigInt(Math.floor(Math.sqrt(0.2) * 2 ** 96));
    const { rpc } = rpcWith({
      [`${UNI_V3_FACTORY}:${SEL.getPool}`]: (d) => (has(d, WETH) && feeIs(d, 3000) ? addr(P) : ZERO),
      [`${UNI_V2_FACTORY}:${SEL.getPair}`]: () => ZERO,
      [`${P}:${SEL.liquidity}`]: () => num(1n),
      [`${P}:${SEL.slot0}`]: () => coder.encode(['uint160', 'int24', 'uint16', 'uint16', 'uint16', 'uint8', 'bool'], [sqrt, 0, 0, 0, 0, 0, true]),
    });
    const ok = new QuoteUsdOracle(rpc, eth(2500), gecko(null).fetchFn, {}, noopLog);
    expect(rel((await ok.get(X, 18, 0))!, 500)).toBeLessThan(1e-6);
    const noEth = new QuoteUsdOracle(rpc, eth(null), gecko(null).fetchFn, {}, noopLog);
    expect(await noEth.get(X, 18, 0)).toBeNull();
  });

  it('V2 route prices from reserves', async () => {
    const X = '0x0aaa000000000000000000000000000000000002';
    const P = '0x2222222222222222222222222222222222222222';
    const { rpc } = rpcWith({
      [`${UNI_V3_FACTORY}:${SEL.getPool}`]: () => ZERO,
      [`${UNI_V2_FACTORY}:${SEL.getPair}`]: (d) => (has(d, USDG) ? addr(P) : ZERO),
      // X is token0 (18 dec): 1000 X vs 2500 USDG (6 dec) → 2.5 USDG per X
      [`${P}:${SEL.getReserves}`]: () => coder.encode(['uint112', 'uint112', 'uint32'], [1000n * 10n ** 18n, 2500n * 10n ** 6n, 0]),
    });
    const o = new QuoteUsdOracle(rpc, eth(2500), gecko(null).fetchFn, {}, noopLog);
    expect(await o.route(X, 0)).toEqual({ kind: 'v2', pool: P, quoteIs0: true, counter: 'usdg', counterDecimals: 6 });
    expect(await o.get(X, 18, 0)).toBeCloseTo(2.5, 9);
  });

  it('keeps the last price when the read fails', async () => {
    let fail = false;
    const { rpc } = rpcWith({
      [`${UNI_V3_FACTORY}:${SEL.getPool}`]: (d) => (has(d, USDG) && feeIs(d, 3000) ? addr(P_USDG_3000) : ZERO),
      [`${UNI_V2_FACTORY}:${SEL.getPair}`]: () => ZERO,
      [`${P_USDG_3000}:${SEL.liquidity}`]: () => num(1n),
      [`${P_USDG_3000}:${SEL.slot0}`]: () => { if (fail) throw new Error('down'); return SLOT0_AMD; },
    });
    const o = new QuoteUsdOracle(rpc, eth(2500), gecko(null).fetchFn, { ttlMs: 1000 }, noopLog);
    expect(rel((await o.get(AMD, 18, 0))!, 462.81)).toBeLessThan(1e-6);
    fail = true;
    expect(rel((await o.get(AMD, 18, 5000))!, 462.81)).toBeLessThan(1e-6);
  });

  it('falls back to GeckoTerminal only when no pool exists, at most once per minute; null before any success', async () => {
    const { rpc } = rpcWith({ [`${UNI_V3_FACTORY}:${SEL.getPool}`]: () => ZERO, [`${UNI_V2_FACTORY}:${SEL.getPair}`]: () => ZERO });
    const g = gecko('462.81');
    const o = new QuoteUsdOracle(rpc, eth(2500), g.fetchFn, { ttlMs: 1000, geckoTtlMs: 60_000 }, noopLog);
    expect(rel((await o.get(AMD, 18, 0))!, 462.81)).toBeLessThan(1e-9);
    expect(g.count()).toBe(1);
    expect(rel((await o.get(AMD, 18, 30_000))!, 462.81)).toBeLessThan(1e-9); // price cache expired, gecko still within its minute → last value
    expect(g.count()).toBe(1);
    await o.get(AMD, 18, 61_000);
    expect(g.count()).toBe(2);
    const down = new QuoteUsdOracle(rpc, eth(2500), gecko(null, 500).fetchFn, {}, noopLog);
    expect(await down.get(AMD, 18, 0)).toBeNull();
  });
});
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `npx vitest run tests/quoteOracle.test.ts`
Expected: FAIL — cannot resolve `../src/quoteOracle`.

- [ ] **Step 4: Write the oracle**

`src/quoteOracle.ts`:
```ts
import type { Rpc } from './chain/rpc';
import { SEL, WETH, NATIVE, USDG, UNI_V2_FACTORY, UNI_V3_FACTORY, UNI_V3_FEES, GECKO_API } from './chain/constants';
import { encodeCall, word, toUint, toAddress } from './chain/abi';
import { sqrtPriceToQuotePerToken } from './decoders/common';
import type { EthUsdOracle } from './enrich';
import { log as defaultLog, type LogFn } from './logger';

export interface QuoteRoute {
  kind: 'v2' | 'v3';
  pool: string;
  /** The quote asset (e.g. the stock) is token0 of that pool. */
  quoteIs0: boolean;
  counter: 'usdg' | 'weth';
  counterDecimals: number;
}

export interface QuoteOracleOpts {
  ttlMs?: number;       // price cache, default 30 s
  routeTtlMs?: number;  // route cache, default 1 h
  geckoTtlMs?: number;  // min interval between GeckoTerminal requests per quote, default 60 s
}

interface Candidate extends QuoteRoute { depthCall: { method: string; params: unknown[] } }

const COUNTERS: Array<{ name: 'usdg' | 'weth'; addr: string; dec: number }> = [
  { name: 'usdg', addr: USDG, dec: 6 },
  { name: 'weth', addr: WETH, dec: 18 },
];
const isZero = (hex: string) => /^0x0*$/.test(hex);
const rank = (r: QuoteRoute) => (r.counter === 'usdg' ? 2 : 0) + (r.kind === 'v3' ? 1 : 0);

/**
 * USD per whole unit of a quote asset. ETH/WETH → the ETH oracle; USDG → 1; any other ERC-20 → its
 * deepest on-chain Uniswap pool against USDG (else WETH), read every ttlMs. GeckoTerminal is used only
 * when no such pool exists, and at most once per geckoTtlMs per quote.
 */
export class QuoteUsdOracle {
  private prices = new Map<string, { value: number; at: number }>();
  private routes = new Map<string, { route: QuoteRoute | null; at: number }>();
  private geckoAt = new Map<string, number>();
  private warned = new Set<string>();
  private ttlMs: number;
  private routeTtlMs: number;
  private geckoTtlMs: number;

  constructor(
    private rpc: Pick<Rpc, 'call' | 'batchSettled'>,
    private eth: Pick<EthUsdOracle, 'get'>,
    private fetchFn: typeof fetch = fetch,
    opts: QuoteOracleOpts = {},
    private logFn: LogFn = defaultLog,
  ) {
    this.ttlMs = opts.ttlMs ?? 30_000;
    this.routeTtlMs = opts.routeTtlMs ?? 3_600_000;
    this.geckoTtlMs = opts.geckoTtlMs ?? 60_000;
  }

  async get(quote: string, quoteDecimals: number, now: number): Promise<number | null> {
    const q = quote.toLowerCase();
    if (q === NATIVE || q === WETH) return this.eth.get(now);
    if (q === USDG) return 1;
    const cached = this.prices.get(q);
    if (cached && now - cached.at < this.ttlMs) return cached.value;
    let value: number | null = null;
    try {
      const route = await this.route(q, now);
      value = route ? await this.readRoute(route, quoteDecimals, now) : await this.gecko(q, now);
    } catch (e) {
      this.logFn('warn', `quoteOracle: price for ${q} failed: ${(e as Error).message}`);
    }
    if (value !== null && value > 0 && Number.isFinite(value)) {
      this.prices.set(q, { value, at: now });
      return value;
    }
    return cached?.value ?? null;
  }

  /** Find (and cache) the pool to price `quote` from. Null when no USDG/WETH pool exists. Public for tests. */
  async route(quote: string, now: number): Promise<QuoteRoute | null> {
    const c = this.routes.get(quote);
    if (c && now - c.at < this.routeTtlMs) return c.route;

    const calls = COUNTERS.flatMap((ct) => [
      { method: 'eth_call', params: [{ to: UNI_V2_FACTORY, data: encodeCall(SEL.getPair, ['address', 'address'], [quote, ct.addr]) }, 'latest'] },
      ...UNI_V3_FEES.map((fee) => ({ method: 'eth_call', params: [{ to: UNI_V3_FACTORY, data: encodeCall(SEL.getPool, ['address', 'address', 'uint24'], [quote, ct.addr, fee]) }, 'latest'] })),
    ]);
    const res = await this.rpc.batchSettled(calls);
    const found: Candidate[] = [];
    COUNTERS.forEach((ct, ci) => {
      const base = ci * (1 + UNI_V3_FEES.length);
      const quoteIs0 = quote < ct.addr;
      const v2 = res[base];
      if (v2.ok && typeof v2.result === 'string' && !isZero(v2.result)) {
        const pool = toAddress(v2.result.slice(2));
        found.push({ kind: 'v2', pool, quoteIs0, counter: ct.name, counterDecimals: ct.dec, depthCall: { method: 'eth_call', params: [{ to: pool, data: SEL.getReserves }, 'latest'] } });
      }
      UNI_V3_FEES.forEach((_, fi) => {
        const r = res[base + 1 + fi];
        if (r.ok && typeof r.result === 'string' && !isZero(r.result)) {
          const pool = toAddress(r.result.slice(2));
          found.push({ kind: 'v3', pool, quoteIs0, counter: ct.name, counterDecimals: ct.dec, depthCall: { method: 'eth_call', params: [{ to: pool, data: SEL.liquidity }, 'latest'] } });
        }
      });
    });

    let best: QuoteRoute | null = null;
    if (found.length) {
      const depths = await this.rpc.batchSettled(found.map((f) => f.depthCall));
      let bestKey = -1, bestDepth = -1n;
      for (let i = 0; i < found.length; i++) {
        const f = found[i], d = depths[i];
        if (!d.ok || typeof d.result !== 'string' || d.result.length < 66) continue;
        const depth = f.kind === 'v3' ? toUint(word(d.result, 0)) : toUint(word(d.result, f.quoteIs0 ? 1 : 0)); // v3 liquidity; v2 counter-side reserve
        if (depth === 0n) continue;
        const key = rank(f);
        if (key > bestKey || (key === bestKey && depth > bestDepth)) {
          bestKey = key; bestDepth = depth;
          best = { kind: f.kind, pool: f.pool, quoteIs0: f.quoteIs0, counter: f.counter, counterDecimals: f.counterDecimals };
        }
      }
    }
    this.routes.set(quote, { route: best, at: now });
    if (!best && !this.warned.has(quote)) {
      this.warned.add(quote);
      this.logFn('warn', `quoteOracle: no on-chain USDG/WETH pool for ${quote} — falling back to GeckoTerminal`);
    }
    return best;
  }

  private async readRoute(r: QuoteRoute, quoteDecimals: number, now: number): Promise<number | null> {
    let counterPerQuote: number;
    if (r.kind === 'v3') {
      const hex = await this.rpc.call(r.pool, SEL.slot0);
      counterPerQuote = sqrtPriceToQuotePerToken(toUint(word(hex, 0)), r.quoteIs0, quoteDecimals, r.counterDecimals);
    } else {
      const hex = await this.rpc.call(r.pool, SEL.getReserves);
      const res0 = toUint(word(hex, 0)), res1 = toUint(word(hex, 1));
      const quoteRes = r.quoteIs0 ? res0 : res1, counterRes = r.quoteIs0 ? res1 : res0;
      if (quoteRes === 0n) return null;
      counterPerQuote = (Number(counterRes) / 10 ** r.counterDecimals) / (Number(quoteRes) / 10 ** quoteDecimals);
    }
    if (r.counter === 'usdg') return counterPerQuote;
    const ethUsd = await this.eth.get(now);
    return ethUsd === null ? null : counterPerQuote * ethUsd;
  }

  private async gecko(quote: string, now: number): Promise<number | null> {
    const last = this.geckoAt.get(quote);
    if (last !== undefined && now - last < this.geckoTtlMs) return null;
    this.geckoAt.set(quote, now);
    const res = await this.fetchFn(`${GECKO_API}/tokens/${quote}`, { headers: { accept: 'application/json' }, signal: AbortSignal.timeout(10_000) });
    if (!res.ok) throw new Error(`geckoterminal http ${res.status}`);
    const j = (await res.json()) as any;
    const p = Number(j?.data?.attributes?.price_usd);
    return Number.isFinite(p) && p > 0 ? p : null;
  }
}
```

- [ ] **Step 5: Run the tests and typecheck**

Run: `npx vitest run && npx tsc --noEmit`
Expected: 120 tests PASS (114 + 6); typecheck clean.

- [ ] **Step 6: Commit**

```bash
git add src/quoteOracle.ts src/chain/constants.ts tests/quoteOracle.test.ts tests/abi.test.ts
git commit -m "feat: on-chain quote→USD oracle with GeckoTerminal fallback

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 4: Enrich, card, pipeline and wiring use the quote asset

**Files:**
- Modify: `src/enrich.ts`, `src/card.ts`, `src/pipeline.ts`, `src/index.ts`, `tests/enrich.test.ts`, `tests/card.test.ts`, `tests/pipeline.test.ts`
- Test: `tests/enrich.test.ts`, `tests/card.test.ts`

**Interfaces:**
- Consumes: `QuoteUsdOracle` (Task 3), `PONS_BUY_AMD`/`PONS_POOL_AMD` (Task 2).
- Produces: `EnrichedBuy.quoteUsd: number | null` (replaces `ethUsd`); `EnrichDeps.quoteOracle: Pick<QuoteUsdOracle, 'get'>` (replaces `oracle`); `fmtQuote(raw: bigint, decimals: number, symbol: string): string` (replaces `fmtEth`).

- [ ] **Step 1: Write the failing tests**

`tests/enrich.test.ts` — change the import to `import { EthUsdOracle, enrich, sqrtToEthUsd } from '../src/enrich'; import { QuoteUsdOracle } from '../src/quoteOracle';`, add a helper after `rpcWith`:
```ts
/** Real quote oracle over the fake RPC (ETH quotes read the USDG/WETH slot0 route; GeckoTerminal always fails). */
const oracleFor = (rpc: Rpc) => new QuoteUsdOracle(rpc, new EthUsdOracle(rpc), (async () => new Response('x', { status: 500 })) as typeof fetch, {}, noopLog);
```
then in every existing `enrich(…)` call replace `{ rpc, oracle: new EthUsdOracle(rpc), logFn: noopLog }` with `{ rpc, quoteOracle: oracleFor(rpc), logFn: noopLog }`, and add:
```ts
  it('AMD-quoted curve buy: usd and market cap use the quote price', async () => {
    const trade = decodeTrade(F.PONS_BUY_AMD, F.PONS_POOL_AMD)!;
    const KITTY: TokenRow = { address: F.PONS_POOL_AMD.token, symbol: 'RYZEN', decimals: 18, totalSupply: 10n ** 27n, ponsCurve: F.PONS_POOL_AMD.id, ponsPhase: 0, createdAt: 0, lastRefresh: 0 };
    const { rpc } = rpcWith({
      [`${F.PONS_POOL_AMD.id}:${SEL.getReserves}`]: () => coder.encode(['uint256', 'uint256'], [17810771601609954000n, 374279227508652000000000000n]),
      [`${F.PONS_POOL_AMD.id}:${SEL.realQuoteReserve}`]: () => num(9300000000000000000n),
      [`${F.PONS_POOL_AMD.id}:${SEL.graduationThreshold}`]: () => num(16665504590959020000n),
      [`${KITTY.address}:${SEL.balanceOf}`]: () => num(0n),
    });
    const b = await enrich(trade, KITTY, { rpc, quoteOracle: { get: async () => 462.81 }, logFn: noopLog }, 0);
    expect(b.quoteUsd).toBe(462.81);
    expect(rel(b.usd!, 1.3513)).toBeLessThan(1e-3);
    expect(rel(b.priceQuotePerToken, 4.7587e-8)).toBeLessThan(1e-3);       // AMD per token from reserves
    expect(rel(b.marketCapUsd!, 1e9 * 4.7587e-8 * 462.81)).toBeLessThan(1e-3);
    expect(b.ponsProgressPct).toBeCloseTo(55.8, 0);
  });
```
`tests/card.test.ts` — change the import to `fmtQuote` instead of `fmtEth`, in the `buy()` helper rename `ethUsd: 2476.07` → `quoteUsd: 2476.07` and in the "unknown USD" test `ethUsd: null` → `quoteUsd: null`; replace the four `fmtEth` assertions with:
```ts
    expect(fmtQuote(195000000000000000n, 18, 'ETH')).toBe('0.195 ETH');
    expect(fmtQuote(9900000000000000n, 18, 'ETH')).toBe('0.0099 ETH');
    expect(fmtQuote(1234567000000000000n, 18, 'ETH')).toBe('1.235 ETH');
    expect(fmtQuote(2000000000000000000n, 18, 'ETH')).toBe('2 ETH');
    expect(fmtQuote(2919745101889322n, 18, 'AMD')).toBe('0.00292 AMD');
    expect(fmtQuote(12500000n, 6, 'USDG')).toBe('12.5 USDG');
```
and add a test:
```ts
  it('renders a stock-quoted buy in the quote asset', () => {
    const t = decodeTrade(F.PONS_BUY_AMD, F.PONS_POOL_AMD)!;
    const kitty: TokenRow = { ...TA, address: F.PONS_POOL_AMD.token, symbol: 'RYZEN', ponsCurve: F.PONS_POOL_AMD.id, ponsPhase: 0 };
    const b: EnrichedBuy = { ...t, buyer: '0xb3e204090f631b47d55b222aeb71e714d9593992', quoteUsd: 462.81, usd: 1.35, marketCapUsd: 22024, holder: { isNew: true, positionIncreasePct: null }, ponsProgressPct: 55.8 };
    const { text } = formatCard(b, kitty, SETTINGS, CFG, 40);
    expect(text).toContain('💵 <b>0.00292 AMD</b> ($1.35)');
    expect(text).toContain('🎯 Pons curve · 56% to graduation');
  });
```
`tests/pipeline.test.ts` — replace the `EthUsdOracle` import and usage: `enrichDeps: { rpc, quoteOracle: { get: async () => null }, logFn: noopLog }` (the "usd unknown" behaviour the test relies on is preserved).

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npx vitest run tests/enrich.test.ts tests/card.test.ts tests/pipeline.test.ts`
Expected: FAIL — `quoteOracle`/`quoteUsd`/`fmtQuote` do not exist.

- [ ] **Step 3: Implement**

`src/enrich.ts`:
- Add `import type { QuoteUsdOracle } from './quoteOracle';`
- `EnrichedBuy`: replace `ethUsd: number | null;` with `/** USD per whole unit of the pool's quote asset (ETH price for ETH pools). */ quoteUsd: number | null;`
- `EnrichDeps`: replace `oracle: EthUsdOracle;` with `quoteOracle: Pick<QuoteUsdOracle, 'get'>;`
- In `enrich`, the first `allSettled` entry becomes `deps.quoteOracle.get(trade.pool.quote, trade.pool.quoteDecimals, now)`; rename `ethUsdR` → `quoteUsdR` and `ethUsd` → `quoteUsd`; then:
```ts
  const usd = quoteUsd === null ? null : (Number(trade.quoteIn) / 10 ** trade.pool.quoteDecimals) * quoteUsd;
  const supply = Number(token.totalSupply) / 10 ** token.decimals;
  const marketCapUsd = quoteUsd === null || !(price > 0) ? null : supply * price * quoteUsd;
```
and the return becomes `{ ...trade, buyer, quoteUsd, usd, marketCapUsd, holder, ponsProgressPct, priceQuotePerToken: price }`. `EthUsdOracle` and `sqrtToEthUsd` stay exported (the quote oracle uses them).

`src/card.ts` — replace `fmtEth` with:
```ts
/** Quote amount: 3 decimals at ≥ 1, 3 significant digits below, trailing zeros trimmed, then the symbol. */
export function fmtQuote(raw: bigint, decimals: number, symbol: string): string {
  const n = Number(raw) / 10 ** decimals;
  return trimZeros(n >= 1 ? n.toFixed(3) : n.toPrecision(3)) + ' ' + symbol;
}
```
and in `formatCard`: `` `💵 <b>${fmtQuote(buy.quoteIn, buy.pool.quoteDecimals, buy.pool.quoteSymbol)}</b>${usdPart}` ``.

`src/pipeline.ts` — import `fmtQuote` instead of `fmtEth`; the log line uses `fmtQuote(buy.quoteIn, pool.quoteDecimals, pool.quoteSymbol)`.

`src/index.ts` — `import { QuoteUsdOracle } from './quoteOracle';`; replace `const oracle = new EthUsdOracle(rpc);` with `const quoteOracle = new QuoteUsdOracle(rpc, new EthUsdOracle(rpc));` and `enrichDeps: { rpc, quoteOracle }`.

- [ ] **Step 4: Run the tests and typecheck**

Run: `npx vitest run && npx tsc --noEmit`
Expected: 122 tests PASS (120 + 2); typecheck clean.

- [ ] **Step 5: Commit**

```bash
git add src/enrich.ts src/card.ts src/pipeline.ts src/index.ts tests/enrich.test.ts tests/card.test.ts tests/pipeline.test.ts
git commit -m "feat: enrich/card/pipeline use the quote asset and its USD price

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 5: Discover stock-quoted Pons tokens; `/add` names the quote; README; live check

**Files:**
- Modify: `src/discovery.ts`, `src/commands/texts.ts`, `src/commands/handler.ts`, `README.md`, `tests/discovery.test.ts`, `tests/commands.test.ts`
- Test: `tests/discovery.test.ts`, `tests/commands.test.ts`

**Interfaces:**
- Produces: `ponsPoolId(token: string, quote: string, fee: number, tickSpacing: number): string` (currencies sorted by address); `DiscoveredPools.quoteSymbol: string` (`'ETH'` unless the token is a Pons launch quoted in an ERC-20); discovery accepts any Pons `pairToken`; `T.addNoPools` and `T.added(symbol, venues)` wording below.

- [ ] **Step 1: Write the failing tests**

`tests/discovery.test.ts`:
1. Add constants after `POOLID`: `const AMD = '0x86923f96303d656e4aa86d9d42d1e57ad2023fdc'; const VLAD = '0x133698a17b7b5c2b981555c56c2b00824f517c1b'; const SPCX = '0x4a0e65a3eccec6dbe60ae065f2e7bb85fae35eea'; const str = (s: string) => coder.encode(['string'], [s]);` (keep the existing `str` if already defined).
2. Update the existing `ponsPoolId` assertion to the new signature: `expect(ponsPoolId(TOKEN, NATIVE, 0, 200)).toBe(POOLID);` and add:
```ts
  it('ponsPoolId sorts currencies by address for ERC-20 quotes', () => {
    expect(ponsPoolId(VLAD, SPCX, 0, 200)).toBe('0xaf31df1ebfb8232747db1e0109df074d9e5cd8cccb52ec974e9ddf7c5da25b43');
    expect(ponsPoolId(SPCX, VLAD, 0, 200)).toBe(ponsPoolId(VLAD, SPCX, 0, 200));
  });
```
3. Add, using the existing `rpcWith`/`geckoFetch`/`erc20`/`noV2`/`noV3` helpers:
```ts
  it('tracks a Pons curve quoted in AMD (phase 0) with the quote symbol/decimals', async () => {
    const rpc = rpcWith({
      ...erc20, ...noV2, ...noV3,
      [`${PONS_V2_FACTORY}:${SEL.getLaunchedToken}`]: () => coder.encode([PONS_LAUNCH_TUPLE], [[TOKEN, CURVE, NATIVE, NATIVE, AMD, 16665504590959020000n, 0, 200, 0, false, 0, 0, 0, 0, true]]),
      [`${AMD}:${SEL.symbol}`]: () => str('AMD'),
      [`${AMD}:${SEL.decimals}`]: () => num(18),
      [`${CURVE}:${SEL.realQuoteReserve}`]: () => num(9300000000000000000n),
      [`${CURVE}:${SEL.graduationThreshold}`]: () => num(16665504590959020000n),
    });
    const t = await new Discovery(rpc, geckoFetch([]), noopLog).discover(TOKEN);
    expect(t.quoteSymbol).toBe('AMD');
    expect(t.ponsProgressPct).toBeCloseTo(55.8, 0);
    expect(t.pools).toEqual([{ id: CURVE, token: TOKEN, kind: 'pons_curve', tokenIs0: false, quote: AMD, fee: null, decimals: 18, quoteSymbol: 'AMD', quoteDecimals: 18 }]);
  });

  it('graduated AMD-quoted token: derives the V4 pool with sorted currencies and admits Gecko pools quoted in AMD', async () => {
    const derived = ponsPoolId(TOKEN, AMD, 0, 200);
    const geckoAmd = '0x' + 'ee'.repeat(32);
    const rpc = rpcWith({
      ...erc20, ...noV2, ...noV3,
      [`${PONS_V2_FACTORY}:${SEL.getLaunchedToken}`]: () => coder.encode([PONS_LAUNCH_TUPLE], [[TOKEN, CURVE, NATIVE, NATIVE, AMD, 0, 0, 200, 0, false, 2, 0, 0, 0, true]]),
      [`${AMD}:${SEL.symbol}`]: () => str('AMD'),
      [`${AMD}:${SEL.decimals}`]: () => num(18),
      [`${UNI_V4_STATE_VIEW}:${SEL.getSlot0}`]: (data) => coder.encode(['uint160', 'int24', 'uint24', 'uint24'], [data.includes(derived.slice(2)) || data.includes(geckoAmd.slice(2)) ? 1n : 0n, 0, 0, 0]),
    });
    const gecko = geckoFetch([
      { dex: 'uniswap-v4-robinhood', id: geckoAmd, name: 'TA / AMD 1%', base: TOKEN, quote: AMD },
      { dex: 'uniswap-v4-robinhood', id: '0x' + 'cc'.repeat(32), name: 'TA / USDG 1%', base: TOKEN, quote: '0x5fc5360d0400a0fd4f2af552add042d716f1d168' }, // unrelated quote → dropped
    ]);
    const t = await new Discovery(rpc, gecko, noopLog).discover(TOKEN);
    expect(t.pools.map((p) => [p.id, p.kind, p.tokenIs0, p.quote, p.quoteSymbol, p.quoteDecimals, p.fee])).toEqual([
      [derived, 'v4', TOKEN < AMD, AMD, 'AMD', 18, 0],
      [geckoAmd, 'v4', TOKEN < AMD, AMD, 'AMD', 18, 10000],
    ]);
  });

  it('skips the curve when the quote decimals cannot be read', async () => {
    const rpc = rpcWith({
      ...erc20, ...noV2, ...noV3,
      [`${PONS_V2_FACTORY}:${SEL.getLaunchedToken}`]: () => coder.encode([PONS_LAUNCH_TUPLE], [[TOKEN, CURVE, NATIVE, NATIVE, AMD, 1, 0, 200, 0, false, 0, 0, 0, 0, true]]),
    });
    await expect(new Discovery(rpc, geckoFetch([]), noopLog).discover(TOKEN)).rejects.toMatchObject({ kind: 'no_pools' });
  });
```
Also update the existing "finds a Pons curve on phase 0" expectation if it asserts `venues`/labels — it does not; leave it. The `no_pools` message text is not asserted in discovery tests.

`tests/commands.test.ts`:
- Add `quoteSymbol: 'ETH'` to the `DISCOVERED` literal (after `ponsProgressPct: 17.6,`).
- Add a test:
```ts
  it('/add names a non-ETH quote asset', async () => {
    discoverImpl = async () => ({ ...DISCOVERED, quoteSymbol: 'AMD', pools: [{ ...DISCOVERED.pools[0], quote: '0x86923f96303d656e4aa86d9d42d1e57ad2023fdc', quoteSymbol: 'AMD' }] });
    await handler.handleUpdate(msg(`/add ${TOKEN}`, ADMIN), 0);
    expect(sent.at(-1).text).toContain('quoted in AMD');
  });
```
- In the "/add error paths" test, change the `no_pools` assertion regex from `/No ETH-quoted pool/` to `/No supported pool/`.

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npx vitest run tests/discovery.test.ts tests/commands.test.ts`
Expected: FAIL — `ponsPoolId` signature, missing `quoteSymbol`, curve rejected for a non-native quote.

- [ ] **Step 3: Implement discovery**

`src/discovery.ts`:
1. `ponsPoolId`:
```ts
/** The Uniswap V4 PoolId a Pons curve graduates into: keccak256(abi.encode(currency0, currency1, fee, tickSpacing, hooks)), currencies sorted by address (native 0x0 first). */
export function ponsPoolId(token: string, quote: string, fee: number, tickSpacing: number): string {
  const a = token.toLowerCase(), b = quote.toLowerCase();
  const [c0, c1] = a < b ? [a, b] : [b, a];
  return keccak256(coder.encode(['address', 'address', 'uint24', 'int24', 'address'], [c0, c1, fee, tickSpacing, PONS_V2_HOOK]));
}
```
2. `DiscoveredPools` gains `quoteSymbol: string;`.
3. In `discoverPools`, declare `let quoteSymbol = 'ETH';` next to the other `let`s, and replace the Pons block (from `if (t.exists) {` to the end of the progress read) with:
```ts
      if (t.exists) {
        ponsCurve = String(t.curve).toLowerCase();
        ponsPhase = Number(t.phase);
        const pairToken = String(t.pairToken).toLowerCase();
        let quote = { addr: NATIVE, symbol: 'ETH', decimals: 18, ok: true };
        if (pairToken !== NATIVE) {
          const [qs, qd] = await this.rpc.batchSettled([
            { method: 'eth_call', params: [{ to: pairToken, data: SEL.symbol }, 'latest'] },
            { method: 'eth_call', params: [{ to: pairToken, data: SEL.decimals }, 'latest'] },
          ]);
          const decOk = qd.ok && typeof qd.result === 'string' && qd.result.length >= 66;
          quote = {
            addr: pairToken,
            symbol: (qs.ok && typeof qs.result === 'string' && decodeAbiString(qs.result)) || pairToken.slice(0, 8),
            decimals: decOk ? Number(toUint(word(qd.result as string, 0))) : 18,
            ok: decOk,
          };
          if (!decOk) this.logFn('warn', `discovery: cannot read decimals of quote ${pairToken} for ${token} — skipping its Pons pools`);
        }
        if (quote.ok) {
          quoteSymbol = quote.symbol;
          const fee = Number(t.poolFee);
          const tokenIs0 = quote.addr === NATIVE ? false : token < quote.addr;
          ponsV4Candidate = { id: ponsPoolId(token, quote.addr, fee, Number(t.tickSpacing)), token, kind: 'v4', tokenIs0, quote: quote.addr, fee, quoteSymbol: quote.symbol, quoteDecimals: quote.decimals };
          allowedQuote = { addr: quote.addr, symbol: quote.symbol, decimals: quote.decimals };
          if (ponsPhase < 2) {
            pools.push({ id: ponsCurve, token, kind: 'pons_curve', tokenIs0: false, quote: quote.addr, fee: null, decimals, quoteSymbol: quote.symbol, quoteDecimals: quote.decimals });
            try {
              const [real, thr] = await this.rpc.batch([
                { method: 'eth_call', params: [{ to: ponsCurve, data: SEL.realQuoteReserve }, 'latest'] },
                { method: 'eth_call', params: [{ to: ponsCurve, data: SEL.graduationThreshold }, 'latest'] },
              ]) as string[];
              const t2 = toUint(word(thr, 0));
              if (t2 > 0n) ponsProgressPct = Math.min(100, Number(toUint(word(real, 0)) * 10000n / t2) / 100);
            } catch (e) {
              this.logFn('warn', `discovery: pons progress for ${token} failed: ${(e as Error).message}`);
            }
          }
        }
      }
```
   with `let allowedQuote: { addr: string; symbol: string; decimals: number } | null = null;` declared next to `ponsV4Candidate`.
4. The V4 call becomes `this.v4Candidates(token, ponsV4Candidate ? [ponsV4Candidate] : [], allowedQuote)`, and `return { pools, ponsCurve, ponsPhase, ponsProgressPct, quoteSymbol };`.
5. `v4Candidates(token, extra, allowedQuote: { addr: string; symbol: string; decimals: number } | null = null)`: replace the filter `if (other !== NATIVE && other !== WETH) continue;` with
```ts
      const isEth = other === NATIVE || other === WETH;
      const isAllowed = allowedQuote !== null && other === allowedQuote.addr;
      if (!isEth && !isAllowed) continue;
```
   and the push with
```ts
      out.push({ id, token, kind: 'v4', tokenIs0: other === NATIVE ? false : token < other, quote: other, fee,
        quoteSymbol: isAllowed && !isEth ? allowedQuote!.symbol : 'ETH', quoteDecimals: isAllowed && !isEth ? allowedQuote!.decimals : 18 });
```
6. `discover()`'s `no_pools` message: `'no supported pool found on Pons or Uniswap'`.

`src/commands/texts.ts`: `addNoPools: '❌ No supported pool found for this token on Pons or Uniswap (V2/V3/V4).'`.
`src/commands/handler.ts` `add()`: build the venue list as
```ts
    const venues = d.pools.map((p) => venueLabel(p, d.ponsProgressPct));
    if (d.quoteSymbol !== 'ETH') venues.push(`quoted in ${escapeHtml(d.quoteSymbol)}`);
    await send(T.added(escapeHtml(d.symbol), venues));
```
`src/index.ts` dry-run seeding needs no change (`upsertPool` already stores the new fields).

- [ ] **Step 4: Run the tests and typecheck**

Run: `npx vitest run && npx tsc --noEmit`
Expected: 127 tests PASS (122 + 4 discovery + 1 commands); typecheck clean.

- [ ] **Step 5: README**

In `README.md`, under "Group usage" replace the sentence "Only ETH/WETH-quoted pools are tracked in this version." with:
```markdown
Pons tokens launched against a tokenized stock (AMD, SPCX, …) or USDG are tracked too: the card shows
the amount in that asset (`💵 0.00292 AMD ($1.35)`) and the USD figure comes from the stock's own
on-chain Uniswap pool against USDG or WETH (read every 30 s); GeckoTerminal is consulted only when no
such pool exists. Plain Uniswap tokens are still tracked on their ETH/WETH pools only.
```

- [ ] **Step 6: Commit**

```bash
git add src/discovery.ts src/commands/texts.ts src/commands/handler.ts README.md tests/discovery.test.ts tests/commands.test.ts
git commit -m "feat: discover and track Pons tokens quoted in tokenized stocks

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

- [ ] **Step 7: Live check (controller runs this; needs `.env`)**

Run: `npm run dry -- --token 0x50d0d0da00ffd195d2d1d2448617ad039855ad2b` for ~90 s.
Expected: `dry: tracking $RYZEN… via pons_curve`, then cards with `💵 <b>0.00… AMD</b> ($…)`, `🎯 Pons curve · N% to graduation`, and a plausible MC.

---

## Follow-ups (out of scope)

- On-chain-only mode (drop the GeckoTerminal fallback entirely once every quote of interest has a pool).
- Uniswap V2/V3 pairs quoted in a stock for non-Pons tokens.
- Show the stock's own USD price on the card (`AMD $462.81`).
