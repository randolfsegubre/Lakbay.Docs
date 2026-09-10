# ADR-0022: Lakbay.AgentDesktop is WPF/MVVM with Unity as the composition root

- **Status:** Accepted
- **Date:** 2026-09-08
- **Repo(s) affected:** `Lakbay.AgentDesktop`

## Context

`Lakbay.AgentDesktop` is a native Windows desktop application by
requirement, not by default choice — call-center seats commonly run a
locked-down desktop image where an installed app tied to CTI/telephony
hardware integration (ADR-0023) is the normal shape, not a browser tab.
.NET's built-in `Microsoft.Extensions.DependencyInjection` is used
everywhere else in this platform (`Lakbay.Cms`, `Lakbay.Booking`,
`Lakbay.AvailabilityApi`) and would be the default choice here too — but
Unity Container is deliberately used instead, specifically because it's a
named requirement on one of the job postings this gap-closing work exists
to address, and a WPF composition root is exactly where a container
choice is visible and testable in isolation, unlike a web app's `Program.cs`
which is somewhat interchangeable regardless of container.

## Decision

- **WPF** (.NET, not .NET Framework — targets the same LTS as the rest of
  the platform) with the **MVVM pattern**: Views (XAML) bind to
  ViewModels; ViewModels never reference a View type directly.
- **Unity Container** (`Unity.Container` NuGet package) as the
  composition root, wired in `App.xaml.cs`. Registrations: ViewModels,
  the `IAgentOpsClient` (HTTP/GraphQL client to `Lakbay.AgentOps`), the
  `ITelephonyBridge` client (ADR-0023), and a `INavigationService`
  abstraction so ViewModels can trigger navigation without referencing
  WPF's `Window`/`Frame` types directly (keeps ViewModels unit-testable
  without a UI thread).
- No code-behind logic beyond View-only concerns (a `DataGrid` selection
  helper, focus management) — anything that touches application state or
  calls a service lives in a ViewModel, resolved through Unity.

## Alternatives considered

- **Prism** (which itself commonly uses Unity or DryIoc under the hood) —
  rejected for this build: Prism adds real value at a larger
  multi-module/multi-window scale than one agent desktop screen needs,
  and using Unity directly keeps the container choice — the actual gap
  being closed — visible rather than hidden behind a framework's default.
  Worth revisiting if `Lakbay.AgentDesktop` grows into multiple
  independently-loadable modules.
- **`Microsoft.Extensions.DependencyInjection`** (matching every other
  repo in the platform) — the technically simpler, more consistent
  choice, and explicitly not chosen here: this repo exists in part to
  demonstrate Unity specifically, and using the same container as
  everywhere else would defeat that purpose while adding no real
  architectural benefit over `Microsoft.Extensions.DependencyInjection`
  for an app this size.
- **WinUI 3 / MAUI** (newer Microsoft UI stacks) — rejected: the target
  requirement names WPF specifically (an existing, real, still-widely-run
  enterprise desktop product), and WPF's maturity is exactly what makes
  it the realistic choice for a call-center seat that will run this app
  for years, not the newest available framework.

## Consequences

- `Lakbay.AgentDesktop.csproj` targets `net10.0-windows` with
  `UseWPF=true` — this repo is the platform's first Windows-only
  deployable; every other repo remains cross-platform.
- ViewModels are unit-testable in isolation (xUnit, no WPF dispatcher
  needed) as long as service dependencies are resolved through
  constructor injection via Unity, never `new`'d directly or resolved via
  a static service locator.
- A future contributor unfamiliar with Unity needs a short onboarding
  note in this repo's own `CLAUDE.md`/`Docs/DEVELOPER_HANDBOOK.md` — it's
  a real but less common choice among current .NET DI containers.
