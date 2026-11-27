---
name: avalonia
description: Guides cross-platform .NET UI work with Avalonia, covering MVVM and data binding, XAML layout and controls, styles and themes, custom controls, ReactiveUI or CommunityToolkit.Mvvm view models, performance with large lists, and packaging for Windows, macOS, Linux, iOS, Android and the browser. Use when building, debugging or upgrading an Avalonia app, when an .axaml file or an Avalonia package is in the project, or when asked to "port this WPF app to Avalonia", "fix this Avalonia binding" or "make this Avalonia list faster". Do NOT use for WPF, WinUI or MAUI apps that do not use Avalonia, for Blazor UI (use blazor-expert), or for general C# questions with no UI (use dotnet-guru).
license: Apache-2.0
metadata:
  version: "3.0.0"
---

# Avalonia

Avalonia is a XAML UI framework for .NET that draws its own controls, so one codebase looks and
behaves the same on Windows, macOS, Linux, iOS, Android and WebAssembly. It is closest to WPF, and
most WPF knowledge carries over. The differences that bite are listed under Traps.

## Versions

Avalonia 12 needs .NET 8 or later and recommends .NET 10, which is the only target it supports for
iOS and Android. It dropped .NET Framework and .NET Standard. A project that must stay on either is
on Avalonia 11, and the advice here mostly still applies. Check the `Avalonia` package version in
the project before answering, and check Avalonia's upgrade notes before moving a project from 11
to 12.

## Which reference to load

Load only what the task needs.

| Task | Reference |
|---|---|
| View models, binding modes, converters, DI, design-time data | `resources/mvvm-databinding.md` |
| Picking a control: layout, input, lists, menus, shapes | `resources/controls-reference.md` |
| Custom and templated controls, virtualisation, rendering cost | `resources/custom-controls-advanced.md` |
| Selectors, themes, light and dark variants, control templates | `resources/styling-guide.md` |
| ReactiveUI commands and observables, animations | `resources/reactive-animations.md` |
| Per-platform services, file access, windowing, packaging | `resources/platform-specific.md` |

## Decisions to make early

**MVVM library.** The Avalonia templates offer CommunityToolkit.Mvvm or ReactiveUI. The toolkit
uses source generators (`[ObservableProperty]`, `[RelayCommand]`) and stays out of the way.
ReactiveUI is the better fit when the UI is genuinely event-driven: search-as-you-type, combined
validation, throttled updates. Do not mix the two in one view model. If the project already uses
one, stay with it.

**Compiled bindings.** Set `x:DataType` on every view and data template. Compiled bindings fail at
build time instead of silently at runtime, and they are faster. A reflection binding is the
exception and should say why.

**Project shape.** One shared project holds views, view models and services. Each platform gets a
thin head project (`.Desktop`, `.Android`, `.iOS`, `.Browser`) that only starts the app and
registers its platform services. Platform code reaches shared code through interfaces registered
in DI, never through `#if` in a view model.

```
MyApp/                 shared: App.axaml, Views/, ViewModels/, Models/, Services/, Styles/
MyApp.Desktop/         Windows, macOS, Linux entry point
MyApp.Android/         optional
MyApp.iOS/             optional
MyApp.Browser/         optional, WebAssembly
```

## Minimal app

```csharp
// MyApp.Desktop/Program.cs
public static void Main(string[] args) =>
    BuildAvaloniaApp().StartWithClassicDesktopLifetime(args);

public static AppBuilder BuildAvaloniaApp() =>
    AppBuilder.Configure<App>()
        .UsePlatformDetect()
        .LogToTrace();
```

```xml
<!-- Views/MainWindow.axaml -->
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:vm="using:MyApp.ViewModels"
        x:Class="MyApp.Views.MainWindow"
        x:DataType="vm:MainWindowViewModel"
        Title="My app" Width="800" Height="600">
    <StackPanel Margin="20" Spacing="10">
        <TextBlock Text="{Binding Greeting}" FontSize="24" FontWeight="Bold" />
    </StackPanel>
</Window>
```

`App.axaml` adds `<FluentTheme />` to `Application.Styles`. Without a theme, controls render with
no template and look broken.

## Traps

- **Panels have no `Padding`.** `StackPanel` and `Grid` take `Margin`; wrap them in a `Border` for
  padding. This is the most common WPF habit that fails to compile.
- **Styles are selectors, not keyed resources.** `<Style Selector="Button.primary">` with
  `Classes="primary"` on the control. WPF's `Style x:Key` with `BasedOn` does not translate one to
  one; load `resources/styling-guide.md`.
- **Lists without virtualisation.** `ItemsControl` with a `StackPanel` renders every item. For
  more than a few hundred rows use `ListBox` or an `ItemsRepeater` with a virtualising panel, and
  keep item templates shallow.
- **Work on the UI thread.** Anything slow goes to `Task.Run` or an async API, and results come
  back with `Dispatcher.UIThread.Post` or through a bound property. A frozen window is nearly
  always this.
- **ReactiveUI command errors.** An unhandled exception in a `ReactiveCommand` is routed to
  `ThrownExceptions`. Subscribe to it, or the app will crash on the first failure.
- **Platform differences are real on mobile and the browser.** File pickers, window management and
  native dialogs differ. Use the `StorageProvider` and `TopLevel` APIs rather than `System.IO`
  paths, which do not exist in the browser sandbox.

## Testing

Test view models as plain classes, with no Avalonia types in them. For UI behaviour, use
`Avalonia.Headless` with xUnit or NUnit: it runs real controls without a window, so it works in CI.
Give each view a design-time `DataContext` so the previewer shows real layout.

## Packaging

| Target | Usual route |
|---|---|
| Windows | MSIX or MSI, or a self-contained single-file exe. ClickOnce is a poor fit. |
| macOS | A signed and notarised `.app` in a DMG |
| Linux | AppImage or Flatpak; a `.deb` for a known distribution |
| iOS, Android | The platform head project, published to the stores |
| Browser | Static hosting of the `.Browser` publish output |

Signing, notarisation and per-platform publish commands are in `resources/platform-specific.md`.
