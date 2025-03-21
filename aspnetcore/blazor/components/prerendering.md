---
title: Prerender ASP.NET Core Razor components
author: guardrex
description: Learn about Razor component prerendering in ASP.NET Core Blazor apps.
monikerRange: '>= aspnetcore-8.0'
ms.author: riande
ms.custom: mvc
ms.date: 11/08/2022
uid: blazor/components/prerender
---
# Prerender ASP.NET Core Razor components

<!-- UPDATE 9.0 Activate after release and INCLUDE is updated

[!INCLUDE[](~/includes/not-latest-version.md)]

-->

<!--
    NOTE: The console output block quotes in this topic use a double-space 
    at the ends of lines to generate a bare return in block quote output.
-->

This article explains Razor component prerendering scenarios for server-rendered components in Blazor Web Apps.
  
Prerendering can improve [Search Engine Optimization (SEO)](https://developer.mozilla.org/docs/Glossary/SEO) by rendering content for the initial HTTP response that search engines can use to calculate page rank.

## Persist prerendered state

Without persisting prerendered state, state used during prerendering is lost and must be recreated when the app is fully loaded. If any state is created asynchronously, the UI may flicker as the prerendered UI is replaced when the component is rerendered.

Consider the following `PrerenderedCounter1` counter component. The component sets an initial random counter value during prerendering in `OnInitialized`. After the SignalR connection to the client is established, the component rerenders, and the initial count value is replaced when `OnInitialized` executes a second time.

`Components/Pages/PrerenderedCounter1.razor`:

```razor
@page "/prerendered-counter-1"
@attribute [RenderModeServer]
@inject ILogger<PrerenderedCounter1> Logger

<PageTitle>Prerendered Counter 1</PageTitle>

<h1>Prerendered Counter 1</h1>

<p role="status">Current count: @currentCount</p>

<button class="btn btn-primary" @onclick="IncrementCount">Click me</button>

@code {
    private int currentCount;
    private Random r = new Random();

    protected override void OnInitialized()
    {
        currentCount = r.Next(100);
        Logger.LogInformation("currentCount set to {Count}", currentCount);
    }

    private void IncrementCount()
    {
        currentCount++;
    }
}
```

<!-- UPDATE 8.0 RC2

@attribute [RenderModeInteractiveServer]

-->

Run the app and inspect logging from the component:

> :::no-loc text="info: BlazorSample.Components.Pages.PrerenderedCounter1[0]":::  
> :::no-loc text="      currentCount set to 41":::  
> :::no-loc text="info: BlazorSample.Components.Pages.PrerenderedCounter1[0]":::  
> :::no-loc text="      currentCount set to 92":::

The first logged count occurs during prerendering. The count is set again after prerendering when the component is rerendered. There's also a flicker in the UI when the count updates from 41 to 92.

To retain the initial value of the counter during prerendering, Blazor supports persisting state in a prerendered page using the <xref:Microsoft.AspNetCore.Components.PersistentComponentState> service (and for components embedded into pages or views of Razor Pages or MVC apps, the [Persist Component State Tag Helper](xref:mvc/views/tag-helpers/builtin-th/persist-component-state-tag-helper)).

To preserve prerendered state, decide what state to persist using the <xref:Microsoft.AspNetCore.Components.PersistentComponentState> service. [`PersistentComponentState.RegisterOnPersisting`](xref:Microsoft.AspNetCore.Components.PersistentComponentState.RegisterOnPersisting%2A) registers a callback to persist the component state before the app is paused. The state is retrieved when the app resumes.

The following example demonstrates the general pattern:

* The `{TYPE}` placeholder represents the type of data to persist.
* The `{TOKEN}` placeholder is a state identifier string.

```razor
@implements IDisposable
@inject PersistentComponentState ApplicationState

...

@code {
    private {TYPE} data;
    private PersistingComponentStateSubscription persistingSubscription;

    protected override async Task OnInitializedAsync()
    {
        persistingSubscription = 
            ApplicationState.RegisterOnPersisting(PersistData);

        if (!ApplicationState.TryTakeFromJson<{TYPE}>(
            "{TOKEN}", out var restored))
        {
            data = await ...;
        }
        else
        {
            data = restored!;
        }
    }

    private Task PersistData()
    {
        ApplicationState.PersistAsJson("{TOKEN}", data);

        return Task.CompletedTask;
    }

    void IDisposable.Dispose()
    {
        persistingSubscription.Dispose();
    }
}
```

The following counter component example persists counter state during prerendering and retrieves the state to initialize the component. 

`Components/Pages/PrerenderedCounter2.razor`:

```razor
@page "/prerendered-counter-2"
@attribute [RenderModeServer]
@implements IDisposable
@inject ILogger<PrerenderedCounter2> Logger
@inject PersistentComponentState ApplicationState

<PageTitle>Prerendered Counter 2</PageTitle>

<h1>Prerendered Counter 2</h1>

<p role="status">Current count: @currentCount</p>

<button class="btn btn-primary" @onclick="IncrementCount">Click me</button>

@code {
    private int currentCount;
    private Random r = new Random();
    private PersistingComponentStateSubscription persistingSubscription;

    protected override void OnInitialized()
    {
        persistingSubscription =
            ApplicationState.RegisterOnPersisting(PersistCount);

        if (!ApplicationState.TryTakeFromJson<int>(
            "count", out var restoredCount))
        {
            currentCount = r.Next(100);
            Logger.LogInformation("currentCount set to {Count}", currentCount);
        }
        else
        {
            currentCount = restoredCount!;
            Logger.LogInformation("currentCount restored to {Count}", currentCount);
        }
    }

    private Task PersistCount()
    {
        ApplicationState.PersistAsJson("count", currentCount);

        return Task.CompletedTask;
    }

    void IDisposable.Dispose()
    {
        persistingSubscription.Dispose();
    }

    private void IncrementCount()
    {
        currentCount++;
    }
}
```

<!-- UPDATE 8.0 RC2

@attribute [RenderModeInteractiveServer]

-->

When the component executes, `currentCount` is only set once during prerendering. The value is restored when the component is rerendered:

> :::no-loc text="info: BlazorSample.Components.Pages.PrerenderedCounter2[0]":::  
> :::no-loc text="      currentCount set to 96":::  
> :::no-loc text="info: BlazorSample.Components.Pages.PrerenderedCounter2[0]":::  
> :::no-loc text="      currentCount restored to 96":::

By initializing components with the same state used during prerendering, any expensive initialization steps are only executed once. The rendered UI also matches the prerendered UI, so no flicker occurs in the browser.

For components embedded into a page or view of a Razor Pages or MVC app, you must add the [Persist Component State Tag Helper](xref:mvc/views/tag-helpers/builtin-th/persist-component-state-tag-helper) with the `<persist-component-state />` HTML tag inside the closing `</body>` tag of the app's layout. **This is only required for Razor Pages and MVC apps.** For more information, see <xref:mvc/views/tag-helpers/builtin-th/persist-component-state-tag-helper>.

`Pages/Shared/_Layout.cshtml`:

```cshtml
<body>
    ...

    <persist-component-state />
</body>
```

## Prerendering guidance

Prerendering guidance is organized in the Blazor documentation by subject matter. The following links cover all of the prerendering guidance throughout the documentation set by subject:

* Fundamentals
  * <xref:Microsoft.AspNetCore.Components.Routing.Router.OnNavigateAsync> is executed *twice* when prerendering: [Handle asynchronous navigation events with `OnNavigateAsync`](xref:blazor/fundamentals/routing#handle-asynchronous-navigation-events-with-onnavigateasync)
  * [Startup: Control headers in C# code](xref:blazor/fundamentals/startup#control-headers-in-c-code)
  * [Handle Errors: Prerendering](xref:blazor/fundamentals/handle-errors#prerendering)
  * [SignalR: Prerendered state size and SignalR message size limit](xref:blazor/fundamentals/signalr#prerendered-state-size-and-signalr-message-size-limit)
* [Render modes: Prerendering](xref:blazor/components/render-modes#prerendering)
* Components
  * [Control `<head>` content during prerendering](xref:blazor/components/control-head-content#control-head-content-during-prerendering)
  * Razor component lifecycle subjects that pertain to prerendering
    * [Component initialization (`OnInitialized{Async}`)](xref:blazor/components/lifecycle#component-initialization-oninitializedasync)
    * [After component render (`OnAfterRender{Async}`)](xref:blazor/components/lifecycle#after-component-render-onafterrenderasync)
    * [Stateful reconnection after prerendering](xref:blazor/components/lifecycle#stateful-reconnection-after-prerendering)
    * [Prerendering with JavaScript interop](xref:blazor/components/lifecycle#prerendering-with-javascript-interop): This section also appears in the two JS interop articles on calling JavaScript from .NET and calling .NET from JavaScript.
  * [QuickGrid component sample app](xref:blazor/components/quickgrid#sample-app): The [**QuickGrid for Blazor** sample app](https://aspnet.github.io/quickgridsamples/) is hosted on GitHub Pages. The site loads fast thanks to static prerendering using the community-maintained [`BlazorWasmPrerendering.Build` GitHub project](https://github.com/jsakamoto/BlazorWasmPreRendering.Build).
  * [Prerendering when integrating components into Razor Pages and MVC apps](xref:blazor/components/integration)
* Authentication and authorization
  * [Server-side threat mitigation: Cross-site scripting (XSS)](xref:blazor/security/server/threat-mitigation#cross-site-scripting-xss)
  * [Unauthorized content display while prerendering with a custom `AuthenticationStateProvider`](xref:blazor/security/server/index#unauthorized-content-display-while-prerendering-with-a-custom-authenticationstateprovider)
  * [WebAssembly prerendering support](xref:blazor/security/webassembly/index#prerendering-support)
<!-- UPDATE 8.0 HOLD LINK FOR WORK AT DESTINATION * [Blazor WebAssembly rendered component authentication with prerendering](xref:blazor/security/webassembly/additional-scenarios#prerendering-with-authentication) -->
* [State management: Handle prerendering](xref:blazor/state-management#handle-prerendering): Besides the *Handle prerendering* section, several of the article's other sections include remarks on prerendering.
