# dotnet pack

**Applies to:** C#/.NET — files matching `*.cs`.

## Null-forgiving `!` overuse

- **Flag:** The null-forgiving operator (`!`) used to silence a nullable-reference-type warning instead of an actual null check, especially chained across a member-access expression.
- **Why:** `!` doesn't make the value non-null — it just tells the compiler to stop warning you. If the value genuinely can be null at runtime (a missing config entry, an unmapped navigation property, a lookup that legitimately misses), you've traded a compile-time warning for a `NullReferenceException` in production, at the exact call site the compiler was trying to protect.
- **Category:** risk
- **Tier:** Major
- **Example:**
  ```csharp
  var displayName = user!.Profile!.PreferredName!;
  ```

## `async void` outside event handlers

- **Flag:** A method declared `async void` that isn't a UI event handler (`onclick`, `EventCallback`, etc.).
- **Why:** Exceptions thrown inside an `async void` method can't be awaited or caught by the caller — they escape straight to the synchronization context (or crash the process on a thread-pool thread). `async Task` gives the caller a faulted task it can observe and handle; `async void` gives them nothing.
- **Category:** risk
- **Tier:** Major
- **Example:**
  ```csharp
  public async void ProcessQueueMessage(Message message)
  {
      await _handler.HandleAsync(message);
  }
  ```

## Sync-over-async

- **Flag:** Blocking on an async call with `.Result`, `.Wait()`, or `.GetAwaiter().GetResult()` instead of awaiting it.
- **Why:** On a context with a captured `SynchronizationContext` (classic ASP.NET, WPF, WinForms) this deadlocks: the blocked thread holds the context while the continuation is queued back onto the same context, waiting for the thread that's blocking it. Even without a deadlock, it wastes a thread-pool thread parked doing nothing instead of returning it to the pool.
- **Category:** risk
- **Tier:** Major
- **Example:**
  ```csharp
  var result = _repository.GetOrderAsync(orderId).Result;
  ```

## Swallowed exceptions / overly broad catch

- **Flag:** A `catch (Exception)` block (or bare `catch`) that does nothing, only logs at a level nobody reads, or continues silently without a rethrow — with no narrower exception type to justify the breadth.
- **Why:** Swallowing an exception turns a loud, diagnosable failure into a silent one. The bug still happens; you've just deleted your only signal that it did, so it resurfaces later as corrupted state or a support ticket with no stack trace to work from.
- **Category:** risk
- **Tier:** Major
- **Example:**
  ```csharp
  try
  {
      await _paymentGateway.ChargeAsync(order);
  }
  catch (Exception)
  {
      // ignored
  }
  ```

## Undisposed `IDisposable` / missing `using`

- **Flag:** An `IDisposable` (`HttpClient` misuse aside, think `FileStream`, `SqlConnection`, `MemoryStream`, a Fluxor/DI-scoped resource, a manually-created `CancellationTokenSource`) constructed without a `using`/`using var`/`try`-`finally` that guarantees disposal.
- **Why:** Undisposed disposables leak the underlying OS handle, connection, or unmanaged buffer. In a long-running service that's a slow handle leak that eventually exhausts the connection pool or file-handle table — the kind of bug that only shows up under load, days after deploy.
- **Category:** risk
- **Tier:** Major
- **Example:**
  ```csharp
  var stream = new FileStream(path, FileMode.Open);
  var bytes = new byte[stream.Length];
  stream.Read(bytes, 0, bytes.Length);
  return bytes;
  ```

## Fire-and-forget `Task` not awaited

- **Flag:** An async call whose returned `Task` is neither awaited nor explicitly assigned/observed (no `await`, no stored reference, no `_ = ...` discard with a documented reason).
- **Why:** An unawaited task's exceptions go unobserved and its completion is untracked — the caller moves on before the work (or its failure) has happened, so ordering bugs and silently-lost work follow. Even when fire-and-forget is intentional, leaving it unmarked reads as a mistake to the next reviewer and hides genuine bugs the same way.
- **Category:** risk
- **Tier:** Minor
- **Example:**
  ```csharp
  public void OnOrderPlaced(Order order)
  {
      _auditLog.RecordAsync(order);
  }
  ```

## Missing `CancellationToken` on async APIs

- **Flag:** A public async method that performs I/O (HTTP call, DB query, file access) but doesn't accept and propagate a `CancellationToken`.
- **Why:** Without a token, callers can't cancel the work when the caller is cancelled — a client that navigates away, an HTTP request that's aborted, or a shutdown signal all keep running the operation to completion anyway, wasting resources and delaying graceful shutdown.
- **Category:** risk
- **Tier:** Minor
- **Example:**
  ```csharp
  public async Task<Order> GetOrderAsync(int orderId)
  {
      return await _db.Orders.FirstAsync(o => o.Id == orderId);
  }
  ```

## LINQ multiple enumeration

- **Flag:** The same `IEnumerable<T>` (not materialized to a `List`/array) enumerated more than once — e.g. a `.Count()` check followed by a `foreach` over the same query, or the same deferred query passed to two LINQ operators.
- **Why:** A deferred LINQ query re-runs its full pipeline (including the underlying DB query, if it's `IQueryable`) every time it's enumerated. Beyond the perf cost, if the source is non-deterministic (a live collection, a random shuffle) each enumeration can silently see different data, producing subtly inconsistent results.
- **Category:** maintainability
- **Tier:** Minor
- **Example:**
  ```csharp
  var activeUsers = _db.Users.Where(u => u.IsActive);
  if (activeUsers.Count() == 0) return;
  foreach (var user in activeUsers) { Notify(user); }
  ```

## Magic strings for config keys

- **Flag:** A configuration key, feature-flag name, or claim type referenced as a raw string literal at each call site instead of a shared constant.
- **Why:** A typo in one of the copies compiles fine and fails silently at runtime (the config lookup just returns the default/missing value), and renaming the key means hunting every literal occurrence instead of one constant with a compiler-checked reference.
- **Category:** maintainability
- **Tier:** Minor
- **Example:**
  ```csharp
  var timeout = _configuration["Services:PaymentGateway:TimeoutSeconds"];
  var retries = _configuration["Services:PaymentGatway:RetryCount"];
  ```

## `== null` and mutable DTOs

- **Flag:** `== null` / `!= null` comparisons instead of the `is null` / `is not null` pattern, and DTOs modelled as mutable classes with settable properties when the shape is really an immutable value.
- **Why:** `is null` can't be intercepted by an overloaded `==` operator, so it's the reliable null check regardless of the type; `== null` can silently call a custom equality operator instead. Mutable DTOs invite update-in-place bugs (a shared reference mutated by one caller unexpectedly affects another) that a `record` with `init`-only properties rules out at compile time.
- **Category:** nit
- **Tier:** Nit
- **Example:**
  ```csharp
  public class OrderSummaryDto
  {
      public int OrderId { get; set; }
      public decimal Total { get; set; }
  }

  if (order == null) return NotFound();
  ```
