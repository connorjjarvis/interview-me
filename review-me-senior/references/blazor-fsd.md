# blazor-fsd pack

**Applies to:** Blazor WebAssembly + MudBlazor + Fluxor + Feature Sliced Design — files matching `*.razor` (and `*.cs` alongside them), or code with `using Fluxor` / `using MudBlazor`.

## Fluxor: mutating state outside a reducer

- **Flag:** Fluxor state mutated directly from a component, service, or effect instead of via `IDispatcher.Dispatch(action)` into a `[ReducerMethod]`.
- **Why:** Fluxor's whole contract is that state only ever changes through a reducer, so every transition is traceable, replayable, and testable in isolation. Mutating it from anywhere else breaks that guarantee — the DevTools history stops matching reality, and two places changing the "same" state race each other with no single source of truth.
- **Category:** design
- **Tier:** Major
- **Example:**
  ```csharp
  public class LeadListService
  {
      private readonly IState<LeadListState> _state;

      public void MarkLeadAsRead(int leadId)
      {
          _state.Value.Leads.First(l => l.Id == leadId).IsRead = true;
      }
  }
  ```

## Fluxor: side effects inside a `[ReducerMethod]`

- **Flag:** A `[ReducerMethod]` that performs I/O — an HTTP call, a `NavigationManager.NavigateTo`, a `Task.Delay`, logging with side effects — instead of just computing and returning the new state.
- **Why:** Reducers must be pure, synchronous functions of `(state, action) -> newState`, because Fluxor may re-run or replay them (time-travel debugging, hot reload). I/O in a reducer breaks that purity, can fire the same network call twice, and can't be awaited by the dispatcher, so failures go unobserved.
- **Category:** design
- **Tier:** Major
- **Example:**
  ```csharp
  [ReducerMethod]
  public static LeadListState ReduceLoadLeadsAction(LeadListState state, LoadLeadsAction action)
  {
      var leads = _httpClient.GetFromJsonAsync<List<Lead>>("api/leads").Result;
      return state with { Leads = leads, IsLoading = false };
  }
  ```

## Fluxor: component reads state directly instead of via `IState<T>`

- **Flag:** A component injecting the store/state object directly and reading its current value instead of injecting `IState<LeadListState>` and binding to `.Value` (or subscribing via `StateChanged`).
- **Why:** `IState<T>` is what wires the component into Fluxor's change notifications — inject anything else and the component reads a snapshot once, then never re-renders when the store updates, so the UI silently goes stale after the first dispatch.
- **Category:** design
- **Tier:** Major
- **Example:**
  ```razor
  @inject IStore Store

  <MudText>@Store.State.GetType().GetProperty("LeadCount")</MudText>
  ```

## FSD: slice boundary violations

- **Flag:** A feature slice importing another feature's internal types/components directly (e.g. `features/lead-editing` reaching into `features/lead-list/Internal/`), or genuinely shared logic duplicated inside a feature instead of promoted to `shared/` or `entities/`.
- **Why:** Feature Sliced Design's whole value is that slices are independently understood, tested, and replaced — a feature reaching into another feature's internals creates a hidden coupling that isn't visible from either slice's public surface, so refactoring or deleting one feature silently breaks the other.
- **Category:** design
- **Tier:** Major
- **Example:**
  ```csharp
  // in features/lead-editing/LeadEditForm.razor.cs
  using HLP.Client.Features.LeadList.Internal;

  var formatter = new LeadListInternalFormatter();
  ```

## Blazor WASM: undisposed subscriptions

- **Flag:** A component that subscribes to an event, a Fluxor `StateChanged`/`ActionSubscriber`, a `NavigationManager.LocationChanged`, or a SignalR hub callback, but doesn't implement `IDisposable` (or `IAsyncDisposable`) to unsubscribe in `Dispose`.
- **Why:** Blazor doesn't unsubscribe event handlers for you when a component is torn down. Every navigation that recreates the component leaves the old subscription alive, so the delegate keeps firing against a disposed component (or just keeps accumulating handlers), leaking memory and eventually double- or triple-invoking logic.
- **Category:** risk
- **Tier:** Major
- **Example:**
  ```razor
  @inject NavigationManager Nav

  @code {
      protected override void OnInitialized()
      {
          Nav.LocationChanged += HandleLocationChanged;
      }
  }
  ```

## Blazor WASM: heavy synchronous work on the UI thread

- **Flag:** Expensive synchronous work — large in-memory sorting/filtering, JSON parsing of a big payload, a tight compute loop — run directly in `OnInitialized`/a render path instead of `OnInitializedAsync`/offloaded, with no yielding.
- **Why:** Blazor WASM runs on a single UI thread inside the browser's WebAssembly runtime — there's no background thread pool to absorb the work. Synchronous CPU-bound work on that thread blocks rendering and input entirely, so the app visibly freezes for however long the work takes.
- **Category:** risk
- **Tier:** Major
- **Example:**
  ```csharp
  protected override void OnInitialized()
  {
      var sorted = _allLeads.OrderBy(l => l.Score).ThenBy(l => l.Name).ToList();
      _rankedLeads = ComputeRankings(sorted);
  }
  ```

## Blazor WASM: missing `@key` on a rendered list

- **Flag:** A `@foreach` rendering a list of components/elements with no `@key` on the outer element, especially when the list is reorderable, filterable, or has items added/removed.
- **Why:** Without `@key`, Blazor's diffing algorithm matches old and new render trees by position, not identity — reordering or removing an item can make Blazor reuse the wrong element's component state (an open dropdown, a bound input value) against the wrong data row.
- **Category:** maintainability
- **Tier:** Minor
- **Example:**
  ```razor
  @foreach (var lead in _leads)
  {
      <LeadRow Lead="lead" />
  }
  ```

## Blazor WASM: giant components and parameter mutation

- **Flag:** A single `.razor` component mixing markup, state, and business logic well past a screen's worth of code (a good sign is a `@code` block doing orchestration that belongs in a slice's feature logic), a `[Parameter]` value mutated inside the component that owns it, or a hand-rolled `StateHasChanged()` call papering over a binding that should just work.
- **Why:** A component that's grown into "the page plus all its logic" can't be tested or reused independently and becomes the one file everyone's afraid to touch. Mutating a `[Parameter]` breaks the one-way data flow Blazor is built around — the parent's copy and the child's mutated copy diverge silently. A `StateHasChanged()` call usually means a data-binding or Fluxor subscription is missing, not that the framework needs a manual nudge.
- **Category:** maintainability
- **Tier:** Minor
- **Example:**
  ```razor
  @code {
      [Parameter] public LeadSummary Lead { get; set; } = default!;

      private void ToggleStar()
      {
          Lead.IsStarred = !Lead.IsStarred;
          StateHasChanged();
      }
  }
  ```

## MudBlazor: bypassing the theme system

- **Flag:** Inline `Style="..."` attributes hard-coding colors/spacing on MudBlazor components instead of `Class`/theme tokens, a `<MudForm>` with inputs that have no `Required`/`Validation` wired up, or a hand-rolled `<input>`/`<select>`/`<button>` where a `MudTextField`/`MudSelect`/`MudButton` would give the same result for free.
- **Why:** Inline styles fight the theme — a dark-mode toggle or a brand-color change won't reach them, and they're invisible to anyone auditing the design system for consistency. A `MudForm` without validation lets the "form" report `IsValid == true` regardless of what's in it, so the enforcement the component exists to provide never runs. Hand-rolled controls silently lose the accessibility, theming, and keyboard behavior MudBlazor already solved.
- **Category:** maintainability
- **Tier:** Nit
- **Example:**
  ```razor
  <MudForm @ref="_form">
      <input @bind="_lead.Email" style="border: 1px solid #ccc; padding: 4px;" />
  </MudForm>
  ```
