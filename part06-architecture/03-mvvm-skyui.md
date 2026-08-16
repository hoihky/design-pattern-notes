---
title: MVVM in SkyUI
order: 3
---

# MVVM in SkyUI

SkyUI is an Avalonia UI toolkit. Its **SkyUI.Demo** gallery and **SkyUI.Data** controls demonstrate MVVM at two scales: full demo pages with `Models/` + `ViewModels/` + `Views/` folders, and embedded ViewModels inside reusable controls.

## Project Layout — The Canonical MVVM Folder Structure

```
skyui/src/SkyUI.Demo/
├── Models/           # Demo data objects (DemoCheckedListNode, DemoTreeData)
├── ViewModels/       # One ViewModel per demo page
└── Views/Demos/      # XAML + thin code-behind
```

This three-folder split is the teaching layout for MVVM: **Model** holds data shape, **ViewModel** holds presentation state and commands, **View** holds markup.

---

## Example 1: Filter Editor — Production Control MVVM

The filter editor is the richest MVVM example in SkyUI — a reusable control with its own ViewModel, commands, live SQL preview, and tree binding.

### The Problem

Users build nested AND/OR filter trees (field conditions inside groups). The UI must:

- Add/remove groups and conditions via toolbar buttons
- Bind a tree view to the document structure
- Show live SQL as the user edits
- Stay reusable across apps without copying logic into every page

### Layer Breakdown

| Layer | Type | Role |
|-------|------|------|
| **Model** | `FilterDocument` | Plain document: root group, field list — no Avalonia types |
| **Model (tree)** | `FilterGroupNode`, `FilterConditionNode` | Bindable tree nodes with `INotifyPropertyChanged` |
| **ViewModel** | `FilterEditorViewModel` | Commands, selection, SQL preview, wires tree change events |
| **View** | `FilterEditor` control + `FilterEditor.axaml` theme | Sets `DataContext`, hosts bindings |

### FilterEditorViewModel — What It Does

The source documents its role explicitly: *"MVVM surface for FilterEditor: commands, selection, SQL preview, and live wiring to the document tree."*

On construction it:

1. Stores the `FilterDocument` and `IFilterSqlExporter` (Strategy for SQL dialect)
2. Creates five `ICommand` instances via `FilterEditorRelayCommand`
3. Subscribes to `Fields.CollectionChanged` and recursively wires `PropertyChanged` on every node in the tree
4. Calls `RefreshSql()` to populate `SqlPreview`

```csharp
AddAndGroupCommand = new FilterEditorRelayCommand(() => AddGroup(FilterLogicalKind.And));
AddOrGroupCommand = new FilterEditorRelayCommand(() => AddGroup(FilterLogicalKind.Or));
AddConditionCommand = new FilterEditorRelayCommand(AddCondition);
RemoveSelectedCommand = new FilterEditorRelayCommand(RemoveSelected, CanRemoveSelected);
RemoveNodeCommand = new FilterEditorNodeCommand(RemoveNode);
```

**`SqlPreview`** is a read-only bindable property. When any node property changes, `OnNodePropertyChanged` calls `RefreshSql()`, which runs the Visitor-based exporter (Part 4) and updates the preview string.

**`SelectedNode`** tracks tree selection so Remove operates on the right node.

### FilterEditor Control — View Responsibilities

The `FilterEditor` control (View) does **not** contain filter logic. When the `Document` property changes:

1. Disposes the old ViewModel (unsubscribes tree events — prevents memory leaks)
2. Creates `new FilterEditorViewModel(document)`
3. Sets `DataContext` on the control root

XAML in the theme binds toolbar buttons:

```xml
<Button Content="+ AND group" Command="{Binding AddAndGroupCommand}" />
<Button Content="Remove selected" Command="{Binding RemoveSelectedCommand}" />
```

The tree binds to `Document.Root.Children`. Condition editors bind to node properties on selected items.

### Walkthrough: User Clicks "+ AND group"

1. Button `Command` invokes `AddAndGroupCommand.Execute`
2. ViewModel `AddGroup(And)` creates `FilterGroupNode`, adds to parent's children
3. `WireFilterGroup` subscribes to new node's events
4. `RefreshSql()` runs `IFilterSqlExporter.Export(document.Root)`
5. `SqlPreview` setter raises `PropertyChanged`
6. Bound `TextBlock` in View updates SQL text

**No code-behind click handler** — pure MVVM command flow.

---

## Example 2: Forms Demo — Two-Way Binding and Validation

`FormsDemoViewModel` exposes form fields as bindable properties:

- `Email`, `EmailError`, `SearchQuery`, `Password`, `Theme`, `Volume`
- `ValidateEmailCommand`, `ClearErrorsCommand`
- `EmailValidator` as `ISkyValidator` property for the custom `SkyFormField` control

XAML two-way binds:

```xml
<TextBox Text="{Binding Email}" />
<sky:SkyFormField ErrorMessage="{Binding EmailError}" Validator="{Binding EmailValidator}">
```

**ViewModel responsibility:** run validation on command, set `EmailError` string. **View responsibility:** display error styling when `EmailError` is non-empty.

This separates *validation policy* (ViewModel + `SkyValidators` factory) from *error display* (control template).

---

## Example 3: Primitives Demo — CanExecute and Async Commands

`PrimitivesDemoViewModel` demonstrates command gating:

```csharp
SaveCommand = new RelayCommand(ExecuteSave, () => !IsSaving);

// In IsSaving setter:
((RelayCommand)SaveCommand).RaiseCanExecuteChanged();
```

While `IsSaving` is true, Save button disables. XAML also binds `IsLoading="{Binding IsSaving}"` on a progress indicator.

**Teaching point:** async UI work must update `CanExecute` when busy state changes, or users double-click and fire duplicate requests.

---

## Example 4: CheckedListRowModel — Micro-ViewModel per Row

Virtualized lists recycle visual elements. Each visible row needs bindable state (`IsChecked`, `IsExpanded`, `IsSelected`) that survives recycling.

`CheckedListRowModel` is a **row-level ViewModel**:

- Holds display text and expansion state
- Exposes lazy `ToggleExpandCommand` (only created when row has children)
- `CheckedListBox` creates row models from `ICheckedListItemAdapter` (Adapter pattern, Part 3)

The control binds CheckBox `IsChecked` to row model — not to domain objects directly. Domain objects stay in `ObservableCollection` on the page ViewModel (`ListDemoViewModel`).

---

## Example 5: Pickers Demo — MVVM Without Commands

Not every screen needs `ICommand`. `PickersDemoViewModel` exposes computed read-only properties:

```csharp
public string FormattedDate => SkyPickerFormat.FormatDate(SelectedDate, Culture);
```

When `SelectedDate` or `Culture` changes, setter calls `Notify(nameof(FormattedDate), nameof(FormattedLongDate), ...)`.

**Lesson:** MVVM is primarily about **bindable presentation state**. Commands are for actions; computed properties are for formatted display.

---

## SkyUI MVVM and Other Patterns

| Pattern | Where in SkyUI MVVM |
|---------|---------------------|
| **Command** (GoF) | `FilterEditorRelayCommand`, demo `RelayCommand` |
| **Observer** | `INotifyPropertyChanged`, `CollectionChanged` |
| **Strategy** | `IFilterSqlExporter` swappable in ViewModel |
| **Composite + Visitor** | Filter tree model; SQL export (Part 4, Part 5) |
| **Adapter** | `ICheckedListItemAdapter` for list binding |

---

## What Would Break Without MVVM Here?

- `FilterEditor` logic duplicated in every app that needs filters
- SQL preview updates would require manual `TextChanged` handlers on every field editor
- Virtualized list check state would corrupt when rows recycle
- Demo gallery could not serve as copy-paste reference for control consumers

## Next

[MVVM at Application Scale →](04-mvvm-spark-studio.md)
