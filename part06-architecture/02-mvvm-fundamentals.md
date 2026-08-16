---
title: MVVM Fundamentals
order: 2
---

# MVVM Fundamentals

**Model–View–ViewModel (MVVM)** is a presentation architecture pattern that separates user interface markup from presentation logic and domain data. It became the standard for WPF, Xamarin, Avalonia, and MAUI applications because it makes UI **testable** and **designer-friendly**.

## The Problem MVVM Solves

Without MVVM, views accumulate logic:

```csharp
// Anti-pattern: logic in code-behind
private void SaveButton_Click(object sender, RoutedEventArgs e)
{
    if (string.IsNullOrWhiteSpace(EmailTextBox.Text)) { /* show error */ return; }
    _api.SaveUser(EmailTextBox.Text, NameTextBox.Text);
    StatusLabel.Text = "Saved!";
}
```

Problems:

- **Untestable** — must instantiate controls to test save logic
- **Tight coupling** — view knows API client, validation, and formatting
- **Designer friction** — XAML and C# logic intertwined

MVVM moves logic into a **ViewModel** that the View binds to declaratively.

## The Three Layers

```mermaid
flowchart LR
    VIEW[View - XAML + minimal code-behind]
    VM[ViewModel - presentation state + commands]
    MODEL[Model - domain data / services]
    VIEW -->|bindings| VM
    VM -->|uses| MODEL
```

| Layer | Responsibility | Should NOT |
|-------|----------------|------------|
| **Model** | Domain entities, services, plain data | Reference UI frameworks |
| **View** | Layout, styles, user input capture | Contain business rules |
| **ViewModel** | Expose bindable state; translate user actions | Reference specific controls (`TextBox`) |

**Data flows:** View binds to ViewModel properties. User edits flow **two-way** into ViewModel. ViewModel calls Model/services. Model changes notify ViewModel; ViewModel raises `PropertyChanged`; View updates automatically.

## INotifyPropertyChanged — The Binding Contract

UI frameworks need to know when ViewModel data changes. .NET provides:

```csharp
public interface INotifyPropertyChanged
{
    event PropertyChangedEventHandler? PropertyChanged;
}
```

When `Email` changes, the ViewModel raises:

```csharp
PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(Email)));
```

Bound `TextBox.Text="{Binding Email}"` updates without manual `textBox.Text = ...` in code-behind.

### SetField Helper Pattern

SkyUI and Spark Studio repeat a small helper instead of a shared base class:

```csharp
private string _email = "";
public string Email
{
    get => _email;
    set
    {
        if (SetField(ref _email, value))
            OnPropertyChanged(nameof(EmailError)); // dependent property
    }
}

private bool SetField<T>(ref T field, T value, [CallerMemberName] string? name = null)
{
    if (EqualityComparer<T>.Default.Equals(field, value)) return false;
    field = value;
    OnPropertyChanged(name);
    return true;
}
```

**Why no ViewModelBase?** Each demo ViewModel stays self-contained. Libraries avoid forcing a base class on consumers.

## ICommand — Encapsulating User Actions

Buttons need executable actions decoupled from event handlers. WPF/Avalonia expose `ICommand`:

```csharp
public interface ICommand
{
    bool CanExecute(object? parameter);
    void Execute(object? parameter);
    event EventHandler? CanExecuteChanged;
}
```

ViewModel exposes commands as properties:

```csharp
public ICommand SaveCommand { get; }
```

XAML binds:

```xml
<Button Content="Save" Command="{Binding SaveCommand}" />
```

The View does not know *what* save does — only that the ViewModel provides a command.

### RelayCommand — Lightweight Implementation

SkyUI's `FilterEditorRelayCommand` wraps an `Action`:

```csharp
public sealed class FilterEditorRelayCommand : ICommand
{
    private readonly Action _execute;
    private readonly Func<bool>? _canExecute;

    public FilterEditorRelayCommand(Action execute, Func<bool>? canExecute = null)
    {
        _execute = execute ?? throw new ArgumentNullException(nameof(execute));
        _canExecute = canExecute;
    }

    public bool CanExecute(object? parameter) => _canExecute?.Invoke() ?? true;
    public void Execute(object? parameter) => _execute();
    public event EventHandler? CanExecuteChanged;

    public void RaiseCanExecuteChanged() =>
        CanExecuteChanged?.Invoke(this, EventArgs.Empty);
}
```

**CanExecute** disables buttons while async work runs (`IsSaving == true`). **RaiseCanExecuteChanged** tells the UI to re-evaluate button enabled state.

## Data Binding Modes

| Binding | Example | Use case |
|---------|---------|----------|
| One-way | `Text="{Binding Title}"` | Display-only labels |
| Two-way | `Text="{Binding Email}"` | Form fields |
| Command | `Command="{Binding SaveCommand}"` | Buttons, menu items |
| Collection | `ItemsSource="{Binding Items}"` | Lists, trees |

Avalonia supports `x:DataType` for compiled bindings — catch property name typos at build time.

## View Wiring Conventions

**Demo pages** (SkyUI.Demo) set DataContext in code-behind:

```csharp
public FormsDemo()
{
    InitializeComponent();
    DataContext = new FormsDemoViewModel();
}
```

**Applications** (Spark Studio) inject ViewModels via DI:

```csharp
desktop.MainWindow = new MainWindow
{
    DataContext = Services.GetRequiredService<MainWindowViewModel>(),
};
```

DI enables testing ViewModels with fake `IMediator` and services.

## MVVM vs MVC vs MVP

| Pattern | View updates when | Controller/Presenter role |
|---------|-------------------|---------------------------|
| **MVC** | Model changes; View may poll | Controller routes requests |
| **MVP** | Presenter pushes to passive View | Presenter holds all logic |
| **MVVM** | Bindings sync View ↔ ViewModel | ViewModel exposes state; View is active via bindings |

MVVM fits declarative XAML ecosystems best.

## MVVM Inside Controls (Not Just Pages)

Reusable controls can host their own ViewModel:

- `FilterEditor` control creates `FilterEditorViewModel` when `Document` is set
- `CheckedListRowModel` is a micro-ViewModel per virtualized row

The **page** ViewModel and **control** ViewModel can coexist — control exposes bindable surface; page orchestrates navigation.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| ViewModel references `TextBox` | Expose `string Email` property |
| Model implements `INotifyPropertyChanged` for everything | ViewModel wraps Model; Model can stay plain for simple cases |
| Confusing UI `ICommand` with CQRS `IRequest` | UI Command = button action; CQRS Command = application message (Part 6 chapter 7) |
| God ViewModel with 50 properties | Split by screen or feature |

## What Would Break Without MVVM?

- Every UI change requires editing code-behind event handlers
- Unit tests need UI thread and control tree
- Designers cannot work on XAML independently
- Reusing presentation logic across platforms (desktop vs mobile) requires copy-paste

## Next

[MVVM in SkyUI →](03-mvvm-skyui.md)
