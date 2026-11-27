---
name: blazor-expert
description: Guides Blazor development across Blazor Web Apps and their render modes (static SSR, Interactive Server, WebAssembly and Auto), standalone WebAssembly and Hybrid apps. Use when building Blazor components, choosing a render mode, implementing state management, handling routing, JavaScript interop, forms and validation, authentication, or speeding up a slow Blazor app. Do NOT use for a Blazor blog built on Azure Static Web Apps (use blazor-blog-feature), for Avalonia or MAUI UI without Blazor, or for general .NET questions with no Blazor component (use dotnet-guru).
license: Apache-2.0
metadata:
  version: "3.0.0"
---

# Blazor

Blazor builds web UI from Razor components written in C#. Since .NET 8 the starting point is the
Blazor Web App, where each component chooses where it runs. Most confusion in a new app comes from
that choice, so settle it first.

## Render modes (.NET 8 and later)

New projects start from the Blazor Web App template, not the separate Server and WebAssembly
templates. The hosting choice is made per component through its render mode:

| Render mode | Where it runs | Interactive | Use for |
|---|---|---|---|
| Static SSR (the default) | Server, HTML only | No | Content pages, forms posted with enhanced form handling |
| `InteractiveServer` | Server, over SignalR | Yes | Line-of-business UI next to the data |
| `InteractiveWebAssembly` | Browser | Yes | Offline-capable or client-heavy components |
| `InteractiveAuto` | Server first, then WebAssembly once downloaded | Yes | Fast first load with a client-side steady state |

- Set a mode on one component with `@rendermode InteractiveServer`, or for the whole app on
  `<Routes>` and `<HeadOutlet>` in `App.razor`.
- With no render mode, a component is static: `@onclick` handlers do nothing. This is the most
  common "my button does not work" cause in a new app.
- Prerendering is on by default for interactive modes, so `OnInitializedAsync` runs twice. Guard
  expensive calls or persist state across the prerender.
- `[StreamRendering]` sends the page shell first and fills it in as async work completes.
- A WebAssembly or Auto component lives in the `.Client` project and cannot reach server-only
  services directly. Give it an interface with a server implementation and an HTTP one.

Standalone WebAssembly (no server, for static hosting) and Blazor Hybrid (components inside MAUI,
WPF or WinForms through a `BlazorWebView`) are still supported and use the same component model.

## Which reference to load

| Task | Reference |
|---|---|
| Component structure, lifecycle, parameters, cascading values, `RenderFragment` | `resources/components-lifecycle.md` |
| Local and shared state, `EventCallback`, binding, service-based state | `resources/state-management-events.md` |
| Routes, constraints, `NavigationManager`, query strings, layouts | `resources/routing-navigation.md` |
| `EditForm`, input components, DataAnnotations and custom validation | `resources/forms-validation.md` |
| Auth setup, `AuthorizeView`, `[Authorize]`, policies, claims | `resources/authentication-authorization.md` |
| Rendering cost, `Virtualize`, JS interop, lazy loading, AOT and trimming | `resources/performance-advanced.md` |

## Traps

- **The component does not re-render.** A mutated list or object keeps the same reference, and an
  event from outside Blazor (a timer, a service event) does not trigger a render. Reassign, or call
  `InvokeAsync(StateHasChanged)` from the non-UI callback.
- **A child's parameter stops updating.** A child that copies a parameter into a field on
  `OnInitialized` never sees the next value. Read it in `OnParametersSet`, or use it directly.
- **JS interop throws on load.** The DOM does not exist during prerendering. Call JS in
  `OnAfterRenderAsync` when `firstRender` is true, and load scripts as modules with
  `IJSRuntime.InvokeAsync<IJSObjectReference>("import", "./script.js")`.
- **Long lists are slow.** Use `<Virtualize>` once a list reaches a few hundred rows, and set `@key`
  on items in any list that reorders.
- **Leaks in Server mode.** Each user holds a circuit on the server. Unsubscribe from service
  events and dispose timers in `Dispose`, or every visit leaves an object behind.
- **Authorisation only in the UI.** `AuthorizeView` and `[Authorize]` decide what renders. They do
  not protect an API or a service method. Check again on the server.
- **Lost connection in Server mode.** Provide reconnection UI, and expect circuit state to be lost
  after `DisconnectedCircuitRetentionPeriod`. Anything the user cannot afford to lose belongs in
  storage, not the circuit.

## Defaults worth keeping

Prefer `EventCallback` over `Action` for child-to-parent events: it awaits the handler and renders
the parent afterwards. Mark required parameters with `[EditorRequired]`. Use scoped services for
state shared across a page or user, and remember that "scoped" means per circuit in Server mode
and per tab in WebAssembly. For WebAssembly in production, turn on trimming and consider AOT for
CPU-heavy code; both are in `resources/performance-advanced.md`, which is worth reading before a
production release.
