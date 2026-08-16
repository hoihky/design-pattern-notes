---
title: MVVM at Application Scale
order: 4
---

# MVVM at Application Scale — Spark Studio

**Spark Studio** is an Avalonia desktop editor for the Spark game engine. It demonstrates MVVM beyond demo pages: a dedicated **ViewModels project**, dependency injection, async hierarchy loading, and integration with **LightMediator** for use-case dispatch.

## The Problem at Application Scale

A game editor shell must:

- Show project title, hierarchy tree, and selection state
- Load entity hierarchy from the engine session asynchronously
- Route user actions (select entity, refresh) without bloating `MainWindow.axaml.cs`
- Remain testable — ViewModel unit tests without launching Avalonia

Demo-style `new ViewModel()` in every window does not scale. Applications need **composition roots** and **injected services**.

## Project Structure

```
spark-studio/
├── src/SparkStudio/              # Views (MainWindow.axaml), App startup
├── src/SparkStudio.ViewModels/     # ViewModels only — no Avalonia references in ideal case
└── src/SparkStudio.Application/  # Services, DTOs (if present)
```

Separating ViewModels into its own assembly enforces: **presentation logic does not reference XAML types**.

## Composition Root — App.axaml.cs

Spark Studio registers services and resolves `MainWindowViewModel` from DI:

```csharp
services.AddSingleton<MainWindowViewModel>();
Services = services.BuildServiceProvider();

desktop.MainWindow = new MainWindow
{
    DataContext = Services.GetRequiredService<MainWindowViewModel>(),
};
```

**Why singleton ViewModel for main window?** One editor shell per process; hierarchy state persists across interactions. Scoped or transient would reset tree on every resolve.

## MainWindowViewModel — Responsibilities

`MainWindowViewModel` is the presentation brain of the editor shell:

| Concern | How ViewModel handles it |
|---------|--------------------------|
| Project title | Bindable `ProjectTitle` property |
| Hierarchy data | `ObservableCollection<HierarchyNodeDto> HierarchyNodes` |
| Selection | `SelectedEntityId` — setter triggers async `SelectEntityAsync` |
| Engine access | Injected `ISparkEngineSession`, `HierarchyService` |
| Use-case dispatch | Injected `IMediator` for commands (CQRS chapter 8) |

### INotifyPropertyChanged — SetField Pattern

Same idiom as SkyUI demos:

```csharp
private bool SetField<T>(ref T field, T value, [CallerMemberName] string? propertyName = null)
{
    if (EqualityComparer<T>.Default.Equals(field, value)) return false;
    field = value;
    OnPropertyChanged(propertyName);
    return true;
}
```

### Async Side Effects in Property Setters

When user selects a hierarchy node:

```csharp
public string? SelectedEntityId
{
    get => _selectedEntityId;
    set
    {
        if (!SetField(ref _selectedEntityId, value)) return;
        _ = SelectEntityAsync(value); // fire-and-forget with error handling inside
    }
}
```

**Design choice:** selection is bindable state; side effect (tell engine which entity is selected) follows automatically. Alternative: explicit `SelectEntityCommand` — more testable, more ceremony.

Spark Studio uses a **hybrid**: some buttons still use `Click=` in XAML delegating to ViewModel methods — partial migration toward full command binding.

## View Bindings — MainWindow.axaml

```xml
<TextBlock Text="{Binding ProjectTitle}" />
<ListBox ItemsSource="{Binding HierarchyNodes}"
         SelectedValue="{Binding SelectedEntityId}"
         SelectedValueBinding="{Binding Id}" />
```

View contains **layout and styling only**. No `ListBox.SelectionChanged` handler required for basic selection sync — two-way binding propagates `SelectedEntityId`.

## ViewModel Dependencies — Clean Architecture Boundary

`MainWindowViewModel` constructor receives abstractions:

- `IMediator` — application messaging (not `HttpClient`, not engine DLL imports)
- `HierarchyService` — loads tree DTOs
- `ISparkEngineSession` — bridge to native engine

The ViewModel **never** references `MainWindow`, `ListBox`, or Vulkan. It exposes strings and collections the View can bind.

This is MVVM **plus** Dependency Inversion (Part 1): ViewModel depends on ports, composition root wires concretions.

## Testing Strategy

With DI, tests construct ViewModel with fakes:

```csharp
var vm = new MainWindowViewModel(fakeMediator, fakeHierarchy, fakeSession);
vm.SelectedEntityId = "entity-1";
// assert fakeSession.SelectedId == "entity-1"
```

Without MVVM, tests would automation-drive UI controls.

## MVVM + CQRS Preview

When editor actions grow (save scene, import asset), ViewModel methods become thin:

```csharp
await _mediator.SendAsync(new SaveSceneCommand(_currentSceneId), ct);
```

ViewModel stays presentation-focused; handler contains use-case logic. Full flow in chapter 8.

## Hybrid Patterns — Honest Assessment

Spark Studio is not 100% pure MVVM:

| Pure MVVM | Spark Studio today |
|-----------|-------------------|
| All actions via `ICommand` | Some `Click` handlers in code-behind |
| Zero logic in View | Minimal wiring in `MainWindow.axaml.cs` |
| ViewModel never async in setter | `SelectedEntityId` triggers async select |

**Lesson:** production apps migrate incrementally. MVVM is a direction, not a purity test.

## Comparison: SkyUI Demo vs Spark Studio

| Aspect | SkyUI.Demo | Spark Studio |
|--------|------------|--------------|
| ViewModel creation | `new` in code-behind | DI singleton |
| Services | None (self-contained demos) | Mediator, engine session, hierarchy |
| Scope | Control gallery | Full application shell |
| Testability | Demo-level | ViewModel unit tests viable |

## Next

[Clean Architecture Fundamentals →](05-clean-architecture.md)
