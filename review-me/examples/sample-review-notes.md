> Saved to ~/.review-me/<repo-name>/ — not committed to your repo.

# Review — feature/response-cache-layer — 2026-07-07

**Base:** main
**Focus:** cache invalidation correctness, and the auth check on the admin purge endpoint

**Summary:** 10 points — 2 Blocker, 5 Major, 2 Minor, 1 Nit · by category: security 1, correctness 1, risk 2, design 2, tests 1, maintainability 1, question 1, nit 1. 1 additional point filtered as known-OK per `rules.md` (`QuoteService` returning `Result<T>` instead of throwing on a not-found lookup).

## Review map

1. `src/cache/CacheInvalidationController.cs:37` · security · Blocker · `PurgeAll` has no `[Authorize]` attribute — any caller that reaches the route can flush the production cache.
2. `src/cache/QuoteCache.cs:104` · correctness · Blocker · Cache key is built from `productId` only, omitting `tenantId`, so tenant B can read tenant A's cached quote.
3. `src/cache/RedisCacheProvider.cs:61` · risk · Major · No connect/sync timeout configured on the Redis multiplexer, so a degraded Redis instance blocks request threads instead of failing fast.
4. `src/cache/QuoteCache.cs:128` · risk · Major · `[dotnet]` `entry.Pricing!.Total!` null-forgives twice on a value that can genuinely be null when a cache entry is read mid-population.
5. `src/cache/QuoteCacheDecorator.cs:15` · design · Major · Decorator re-implements retry-on-timeout logic that already lives in the `HttpClientFactory` policy — duplicate cross-cutting concern.
6. `src/features/cache-admin/CacheAdminNotifier.cs:22` · design · Major · `[blazor-fsd]` `MarkPurgeInFlight` mutates `_state.Value.PurgeInFlight` directly instead of dispatching into a `[ReducerMethod]`.
7. `src/cache/RedisCacheProvider.cs:8` · tests · Major · No integration test exercises the cache-miss-then-populate path; coverage stops at a unit test against a mocked `IDistributedCache`.
8. `src/cache/QuoteCache.cs:150` · maintainability · Minor · TTL literal `300` repeated three times across the file — extract to a named constant.
9. `src/cache/RedisCacheProvider.cs:22` · question · Minor · Why Redis over an in-process `MemoryCache`, given this service still deploys as a single instance?
10. `src/cache/QuoteCacheDecorator.cs:5` · nit · Nit · Brace style and field naming (`_cache` vs `cache`) drift from the rest of the file.

## Detail

### 1. `src/cache/CacheInvalidationController.cs:37` — security · Blocker · confidence: certain

```csharp
[HttpPost("purge-all")]
public async Task<IActionResult> PurgeAll()
{
    await _cache.FlushAllAsync();
    return Ok();
}
```

**Observation:** Every other action in this controller carries `[Authorize(Roles = "Admin")]`, but `PurgeAll` has neither the attribute nor a manual role check.

**Why:** A full cache flush under load is itself an incident (every request behind it becomes a cold-cache miss simultaneously), and this route currently only inherits whatever base policy applies to the controller — which, on inspection, is `[Authorize]` with no role restriction. Any authenticated user, not just an admin, can trigger it.

**Draft comment kept:** "This action flushes the entire cache and isn't gated by `[Authorize(Roles = "Admin")]` like the rest of the controller — was that deliberate, or did it get dropped when `PurgeAll` was split out of `CacheAdminController`? As written, any authenticated user can force a full cache flush in prod, which given cold-cache load on `quote-service` is an outage-shaped action, not just an inconvenience."

### 2. `src/cache/QuoteCache.cs:104` — correctness · Blocker · confidence: certain

```csharp
private static string BuildCacheKey(int productId, DateOnly quoteDate)
{
    return $"quote:{productId}:{quoteDate:yyyyMMdd}";
}
```

**Observation:** `BuildCacheKey` never folds in `tenantId`, even though `GetQuoteAsync` (`QuoteService.cs:52`) has `TenantContext.Current` in scope at every call site.

**Why:** Two tenants pricing the same product on the same day collide on this key. Whichever tenant's request populates the cache first "wins," and the other tenant is served that price and terms until the entry expires — a correctness bug that's also a data-leak, since quote terms can differ per tenant contract.

**Draft comment kept:** "`BuildCacheKey` doesn't include `tenantId`, but `GetQuoteAsync` definitely has `TenantContext.Current` available when it calls this. Two tenants quoting the same product on the same day will collide on this key — tenant B gets tenant A's cached quote and pricing until the entry expires. Can we fold `tenantId` in, e.g. `quote:{tenantId}:{productId}:{quoteDate:yyyyMMdd}`?"

### 3. `src/cache/RedisCacheProvider.cs:61` — risk · Major · confidence: hunch

```csharp
services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = configuration["Redis:ConnectionString"];
});
```

**Observation:** No `ConnectTimeout` or `SyncTimeout` is set, so the client falls back to the StackExchange.Redis library defaults, and no `CancellationToken` is threaded through the `GetOrCreateAsync` calls that use this provider.

**Why:** This sits in the hot path for every quote request. Under a Redis blip, calls queue on the library default timeout instead of failing fast, and — since there's no circuit breaker elsewhere in the pipeline (checked `Startup.cs`) — a single degraded Redis instance can back up enough requests to exhaust the thread pool. This is a hunch rather than a certainty: it depends on load and on Redis's actual failure mode, which hasn't been exercised in this diff.

**Draft comment kept:** "Should we set an explicit `ConnectTimeout`/`SyncTimeout` here, and maybe thread a `CancellationToken` through `GetOrCreateAsync`, so a slow or degraded Redis instance fails fast instead of queuing requests on the library default? This is the hot path for every quote, so a Redis blip could look like a full outage without one — worth confirming with a load test either way."

### 4. `src/cache/QuoteCache.cs:128` — risk · Major · confidence: hunch

```csharp
private static Quote MapCachedEntry(CachedQuoteEntry entry)
{
    return new Quote
    {
        ProductId = entry.ProductId,
        Total = entry.Pricing!.Total!,
        ExpiresAt = entry.ExpiresAt
    };
}
```

**Observation:** `[dotnet]` `entry.Pricing!.Total!` null-forgives twice on `CachedQuoteEntry.Pricing`, which `RedisCacheProvider.GetOrCreateAsync` (above `BuildCacheKey`) can write to Redis as soon as the key exists — `Pricing` is only populated once the quote engine finishes pricing.

**Why:** `!` doesn't make the value non-null, it just silences the compiler. If `MapCachedEntry` runs against an entry read mid-population — exactly the race point 3's missing timeout makes more likely under load — `Pricing` is genuinely null here and this throws an NRE inside the same hot cache-read path point 3 already flagged as unprotected. Marked as a hunch because it depends on whether a partially-written entry is actually reachable, which isn't exercised in this diff.

**Draft comment kept:** "`entry.Pricing!.Total!` assumes `Pricing` is always populated by the time this reads it, but nothing here guarantees that — can we null-check `Pricing` and treat a null as a cache miss instead of null-forgiving through it? Worth doing alongside the timeout fix in point 3, since a slow Redis read is exactly when a caller could observe a partially-written entry."

### 6. `src/features/cache-admin/CacheAdminNotifier.cs:22` — design · Major · confidence: certain

```csharp
using Fluxor;

public class CacheAdminNotifier
{
    private readonly IState<CacheAdminState> _state;

    public CacheAdminNotifier(IState<CacheAdminState> state) => _state = state;

    public void MarkPurgeInFlight(bool inFlight)
    {
        _state.Value.PurgeInFlight = inFlight;
    }
}
```

**Observation:** `[blazor-fsd]` `MarkPurgeInFlight` writes straight to `_state.Value.PurgeInFlight` on the injected `IState<CacheAdminState>` instead of dispatching an action for a `[ReducerMethod]` to handle.

**Why:** Fluxor's whole contract is that state only changes through a reducer, so every transition is traceable and replayable. `CacheAdminPanel.razor` (which calls `MarkPurgeInFlight` from the purge button) also reads `CacheAdminState` via `IState<T>` — two paths writing the "same" state outside the reducer race each other, and Fluxor's DevTools history stops matching what's actually rendered.

**Draft comment kept:** "This mutates `_state.Value` directly rather than going through a reducer — can we add a `SetPurgeInFlightAction` and a `[ReducerMethod]` for it instead? Right now anything touching `CacheAdminState` outside the reducer can race, which will make the eventual 'why is the spinner stuck' bug much harder to track down."

## Rules added this session

- **expected** — "Cache keys for tenant-scoped data must include `tenantId`." Scope: `src/cache/**`. Rationale: a missing `tenantId` segment caused the cross-tenant cache-poisoning bug found at point 2 of this review. Added: 2026-07-07 (written to `rules.md` on confirmation after drilling into point 2).
