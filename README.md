# Hackathon FAQ: Common Gotchas for Stellar + Anchor + DeFi Apps

This document compiles lessons learned from multiple teams building Stellar applications with Etherfuse, Blend, DeFindex, and other integrations. Use it to avoid the pitfalls we already hit.

---

## Etherfuse (Anchor) Issues

| Gotcha | Solution |
|--------|----------|
| **Bank Account Not Found** (60+ min debugger) | `customer_id` + `bank_account_id` must be generated ONCE, used in onboarding, then reused FOREVER. New IDs = broken. |
| Responses nested under `onramp`/`offramp` keys | Unwrap: `response.onramp \|\| response` |
| Auth header format | `Authorization: ${apiKey}` (no "Bearer" prefix) |
| Sandbox URL different | Use `api.sand.etherfuse.com` not production |
| Order ID field name varies | Check `orderId`, `id`, and `order_id` |
| New orders have indexing delay | Use local state; don't immediately query API |
| Sandbox requires manual fiat simulation | Call `POST /ramp/order/fiat_received` to progress orders |
| Wallet must be registered | Complete Etherfuse dashboard onboarding first |

### The Bank Account ID Issue (Detailed)

**Error**: `Bank account not found`

**What doesn't work**:
- Generating random `bankAccountId` UUIDs on each request
- Registering bank accounts via API
- Using `bankAccountId` from previous onboarding URLs

**Root Cause**: Etherfuse requires exact ID consistency. Every quote and order call must use the exact same `customer_id` + `bank_account_id` pair that was used to generate the onboarding URL the user completed.

**The Fix**:
1. Generate ONE `bankAccountId` UUID and save it
2. Generate the onboarding URL with that specific ID
3. User completes onboarding (this binds the IDs together)
4. Use those SAME IDs for ALL subsequent API calls - forever

---

## Testnet Asset Fragmentation (THE BIG ONE)

| Problem | Impact | Workaround |
|---------|--------|------------|
| Each protocol uses different testnet USDC | DEX USDC ≠ Blend USDC ≠ Etherfuse USDC | Use mock mode for cross-protocol steps |
| No canonical testnet tokens | End-to-end integration testing impossible | Document gaps; judges understand |
| CETES only has mainnet liquidity | `op_no_path` on testnet swaps | Mock swap with realistic rates |

**Key insight**: This works on mainnet because everyone uses Circle's real USDC. Testnet is fragmented by design.

**Example**: You on-ramp MXN → CETES via Etherfuse, swap CETES → USDC on the DEX, but then can't deposit that USDC into Blend because Blend's testnet pool expects a *different* USDC token.

**Solution**: Build with mock/demo modes. Your app flow is correct - it just can't fully execute on testnet. Judges understand this limitation.

---

## Stellar SDK & Network

| Gotcha | Solution |
|--------|----------|
| SDK v14 renamed `SorobanRpc` → `rpc` | Update imports to use `rpc` namespace |
| Text memo max 28 bytes | `uuid.replace(/-/g, "").slice(0, 28)` |
| Trustlines required before receiving | Check and prompt user to add trustline first |
| `sendTransaction` returns PENDING | Must poll for actual ledger inclusion |
| Path payments need `path` param | Empty `path=[]` = direct orderbook only; pass intermediates for multi-hop |
| Polling can hang forever | Add max retry limit (60 attempts / 60 seconds) |
| XDR parsing: `LedgerEntryData` not `LedgerEntry` | Use `xdr.LedgerEntryData.from_xdr()` directly |
| No account-filtered asset listing | Maintain user-managed `tracked_assets` list |

### How DEX Swaps Actually Work

There's rarely direct liquidity between arbitrary pairs (e.g., CETES/USDC). Stellar's pathfinding typically routes through XLM:

```
CETES → XLM → USDC
```

Use `pathPaymentStrictSend` and let Horizon's `/paths/strict-send` endpoint find the route. Don't hardcode paths.

---

## DeFindex Issues

| Gotcha | Solution |
|--------|----------|
| API key required, no self-service | Get key from team or use direct contract calls |
| Endpoint is `/vault/` not `/vaults/` | Check exact path (singular vs plural) |
| Balance param is `?from=` not `?user=` | Test each param name |
| Request body: `{"amounts": [N]}` not `{"amount": N}` | Amounts is array even for single asset |
| Returns HTTP 201, not 200 | Adjust error handling |
| XLM requires SAC wrapping before deposit | Wrap native XLM to Stellar Asset Contract first |
| TVL is at `totalManagedFunds[0].total_amount` | Not a direct `tvl` key |

---

## Blend Protocol Issues

| Gotcha | Solution |
|--------|----------|
| Testnet USDC issuer differs from DEX | Use demo mode; works on mainnet |
| Soroban txs need simulation first | Call `simulateTransaction` before `sendTransaction` |
| Balances in stroops (7 decimals) | Divide by 10,000,000 for display |

---

## Development Environment

| Gotcha | Solution |
|--------|----------|
| macOS port 5000 blocked by AirPlay | Use port 5001 |
| Buffer realm mismatch in Vitest | Route crypto tests to `node` environment via `environmentMatchGlobs` |
| localStorage not available server-side | Check `typeof window !== "undefined"` |
| React Context state isolation | Use module-level state or lift to page component |
| Unfunded account raises `NotFoundError` | Catch separately and show user-friendly message |

---

## Architecture Patterns That Work

| Pattern | Why |
|---------|-----|
| Build with mock/fallback modes from start | Demo always works even when services fail |
| Server-side API routes for anchor calls | Protect API keys from browser exposure |
| Plan before coding | 20 min planning prevents 60 min rebuilds |
| Store IDs in database, not env vars | Each user needs their own anchor IDs |
| Add polling timeouts | Never trust that external status will change |
| Graceful degradation | Disable features cleanly if config missing |

---

## Top 5 Time Wasters (ranked by debug time)

| Rank | Issue | Time Lost | Prevention |
|------|-------|-----------|------------|
| 1 | Etherfuse ID binding | 60+ min | Generate once, reuse forever |
| 2 | Testnet asset fragmentation | Hours | Accept it, use mock mode |
| 3 | Conceptual misalignment | 60 min | Be specific in requirements upfront |
| 4 | Nested API responses | 30 min | Always log and check response shape |
| 5 | Memo length limits | 20 min | Truncate UUIDs from the start |

---

## Who Signs What?

Understanding this prevents confusion:

| Action | Who Signs the Stellar Tx |
|--------|--------------------------|
| On-ramp (receive tokens from anchor) | **Anchor signs** - they send to you |
| Add trustline | **User signs** - authorizing your account |
| Swap on DEX | **User signs** - spending your assets |
| Deposit to DeFi (Blend/DeFindex) | **User signs** - moving your assets |
| Withdraw from DeFi | **User signs** - reclaiming your assets |
| Off-ramp (burn tokens to anchor) | **User signs** - sending to issuer |

---

## Quick Reference: Environment URLs

### Etherfuse
```
Sandbox: https://api.sand.etherfuse.com
Production: https://api.etherfuse.com
```

### Stellar
```
Testnet Horizon: https://horizon-testnet.stellar.org
Testnet Soroban: https://soroban-testnet.stellar.org
Mainnet Horizon: https://horizon.stellar.org
Mainnet Soroban: https://soroban.stellar.org
```

### Asset Issuers (Testnet)
```
CETES: GC3CW7EDYRTWQ635VDIGY6S4ZUF5L6TQ7AA4MWS7LEQDBLUSZXV7UPS4
USDC (Circle testnet): GBBD47IF6LWK7P7MDEVSCWR7DPUWV3NY3DTQEVFL4NAT4AQH3ZLLFLA5
```

### Asset Issuers (Mainnet)
```
USDC (Circle): GA5ZSEJYB37JRC5AVCIA5MOP4RHTM335X2KGX3IHOJAPP5RE34K4KZVN
```

---

## Need Help?

- **Stellar Docs**: [developers.stellar.org](https://developers.stellar.org)
- **Blend Protocol**: [docs.blend.capital](https://docs.blend.capital)
- **Etherfuse**: [etherfuse.com](https://www.etherfuse.com)
- **DeFindex**: Contact team directly for API access

---

*Compiled from: Ya Otter Save, Arroz Wallet, and Stellar DevRel Experiment reports (February 2025)*
