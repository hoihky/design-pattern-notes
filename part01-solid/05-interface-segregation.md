---
title: Interface Segregation Principle
order: 5
---

# Interface Segregation Principle (ISP)

> *Clients should not be forced to depend on interfaces they do not use.*

— Robert C. Martin

ISP fights **fat interfaces** — single types that bundle unrelated operations, forcing every implementer and every consumer to know about methods they'll never call.

## The Core Idea

Split large interfaces into **role-specific** smaller ones. Consumers depend only on the roles they need. Implementers implement only the roles they support (or implement multiple small interfaces).

ISP is the consumer-side mirror of SRP. SRP asks "does this class do too much?" ISP asks "does this interface force clients to know too much?" A perfectly focused class can still expose a fat interface that harms its callers.

### Fat interfaces create hidden coupling

When a client depends on `IImageService` with seven methods, it couples to all seven operation families — even if it calls one. Refactoring one method signature triggers recompilation of unrelated handlers. Mock setup stubs six no-op methods. ISP removes that noise.

## Smell: IGodService

```csharp
interface IImageService
{
    Task ResizeAsync(...);
    Task CropAsync(...);
    Task ApplyFilterAsync(...);
    Task EnhanceAsync(...);
    Task ConvertFormatAsync(...);
    Task ExtractMetadataAsync(...);
    Task GenerateThumbnailAsync(...);
}
```

A thumbnail endpoint depends on the entire interface even though it calls one method. Mocking requires stubbing seven operations. Adding an eighth method forces every mock in the test suite to update — even tests for resize-only handlers.

**Why this violates ISP:** The interface names no client role. It describes the entire image domain, not what a specific caller needs.

## Refactor: Segregate by Client Role

ImgKit splits image operations into three strategy families, each with its own factory. Handlers depend on the narrowest factory that serves their command.

### Three parallel families

| Family | Strategy interface | Factory interface | Example implementations |
|--------|---------------------|-------------------|------------------------|
| Filters | `IImageFilterStrategy` | `IImageFilterStrategyFactory` | Blur, sharpen, grayscale, edge enhance |
| Enhancement | `IImageEnhancementStrategy` | `IImageEnhancementStrategyFactory` | Brightness, contrast |
| Ops | `IImageOpsStrategy` | `IImageOpsStrategyFactory` | Resize, crop, rotate, watermark |

**`IImageFilterStrategy`** — **What it does:** Applies a named PillowNet filter with typed options (`FilterStrategyOptions`: radius, percent, threshold, size). **Why it exists:** Filters share a common `Apply` signature distinct from resize/crop ops. **ISP role:** Filter handlers depend only on filter abstractions.

**`IImageFilterStrategyFactory`** — **What it does:** Resolves strategy by case-insensitive name. **Why it exists:** Handlers should not know all filter types — only how to look one up.

**`ApplyFilterHandler`** (and similar) depends on `IImageFilterStrategyFactory` — not enhancement or ops factories. Each handler's constructor lists only what it uses.

**Problem → Solution → Walkthrough:**

- **Problem:** One mega-interface for all image operations couples every API endpoint to every PillowNet dependency.
- **Solution:** Three strategy + factory pairs; one handler per command type.
- **Walkthrough:**
  1. `ImagesController` receives HTTP request, maps to command (`ApplyFilterCommand`).
  2. LightMediator dispatches to `ApplyFilterHandler`.
  3. Handler injects `IImageFilterStrategyFactory` only.
  4. Factory returns `IImageFilterStrategy` by name.
  5. Handler applies strategy, returns `ProcessedImageResult`.

**What would break without this?** Resize handler would transitively depend on filter and enhancement types. DI registration would require all strategies for every handler. Unit tests for crop would mock filter methods they never call.

## Example 1: Spark — UI Control Interfaces

Spark avoids a monolithic `IUiControl` with every possible widget API. Instead, segregated interfaces inherit a minimal `IUiElement` base.

### Interface hierarchy

**`IUiElement`** — base layout/visibility/hierarchy operations shared by all widgets.

**`IUiButton`** — **What it does:** Label text, click callback, click-state query. **Why it exists:** Button clients need click semantics, not slider range APIs. **ISP role:** Code creating buttons depends on `IUiButton`, not the entire widget surface.

**`ISlider`** — value, range, change callback — nothing about button labels.

**`ILabel`** — text and muted styling — nothing about input or clicks.

```cpp
class IUiButton : public virtual IUiElement {
public:
    virtual void SetLabel(Utf8String label) = 0;
    virtual void SetOnClick(UiVoidCallback handler) = 0;
};

class ISlider : public virtual IUiElement {
public:
    virtual void SetValue(float value) = 0;
    virtual void SetRange(float minValue, float maxValue) = 0;
};

class ILabel : public virtual IUiElement {
public:
    virtual void SetText(Utf8String text) = 0;
    virtual void SetMuted(bool muted) = 0;
};
```

**`IUiControlsFactory`** (Abstract Factory, Part 2) — creates specific control types. Layout code that positions labels never includes slider headers.

**Problem → Solution → Walkthrough:**

- **Problem:** Demo UI code needing a slider would include button/label/checkbox methods via one fat interface.
- **Solution:** Segregated control interfaces + factory returning concrete control type.
- **Walkthrough:**
  1. Demo calls `factory.CreateSlider(parent)`.
  2. Receives `ISlider*` (also an `IUiElement*` for layout).
  3. Configures range and callback — never sees button API.
  4. Layout system positions via `IUiElement` base only.

**What would break without this?** Every UI consumer recompiles when any widget adds a method. Mock UI backends implement dozens of empty stubs. Virtual inheritance complexity explodes if one mega-interface tries to cover ImGui, native, and headless backends.

## Example 2: RainDB — Layered Table Interfaces

Table access splits by capability. Not every catalog consumer needs columnar batch layout.

### `ITableSource` — metadata only

**What it does:** Exposes `TableId`, `Name`, `TableSchema`, and `SchemaVersion`. **Why it exists:** Query planning, catalog listing, and schema validation need table identity without reading data pages. **ISP role:** Metadata clients avoid depending on columnar storage layout.

### `IColumnarTableSource` — scan target

**What it does:** Extends `ITableSource` with `IReadOnlyList<IColumnarBatch> Batches`. **Why it exists:** Vectorized scan, hash aggregate, and join engines read columnar batches. **ISP role:** Scan engines opt into the heavier interface; metadata tools do not.

```csharp
public interface ITableSource
{
    TableId Id { get; }
    string Name { get; }
    TableSchema Schema { get; }
    int SchemaVersion { get; }
}

public interface IColumnarTableSource : ITableSource
{
    IReadOnlyList<IColumnarBatch> Batches { get; }
}
```

**`DefaultQueryExecutor`** pattern-matches physical plans and casts catalog entries to `IColumnarTableSource` only when executing scans/joins/aggregates. A metadata-only code path uses `ITableSource` without touching batches.

### Buffer pool segregation

**`IBufferPool`** — general byte buffer allocation for spill I/O and general-purpose memory.

**`IAlignedBufferPool`** — SIMD-aligned buffers for vectorized kernels (`FixedWidthSelectionKernels`, `ProjectGather`).

Vectorized operators depend on `IAlignedBufferPool`; spill writers use `IBufferPool`. Mixing them in one interface would force spill code to know about SIMD alignment requirements.

**Problem → Solution → Walkthrough:**

- **Problem:** Single `ITable` interface with `GetRows()` forces metadata tools to link columnar batch types.
- **Solution:** Layer `ITableSource` → `IColumnarTableSource`; separate buffer interfaces.
- **Walkthrough:**
  1. SQL compiler binds table name → `TableId` via `ICatalog`.
  2. Physical plan carries `TableId` only.
  3. Executor resolves `ITableSource` from catalog.
  4. Scan plan: cast to `IColumnarTableSource`, pass batches to `VectorizedScanEngine`.
  5. Explain-only plan: reads schema from `ITableSource`, never touches batches.

**What would break without this?** Catalog listing would pull in columnar dependencies. Tests for schema validation would allocate aligned buffers unnecessarily. Interface segregation keeps compile-time dependencies minimal per client.

## Example 3: SkyUI — Validators

SkyUI form validation uses a minimal contract. Complex validation builds by composition, not by inflating the interface.

### `ISkyValidator` — single-rule contract

**What it does:** Accepts a form value (`object?`), returns `SkyValidationResult` (valid/invalid with message). **Why it exists:** Every validation rule — required, email, range, min length — shares this one operation. **ISP role:** Form fields depend on one method, not a validation service with dozens of rule methods.

```csharp
public interface ISkyValidator
{
    SkyValidationResult Validate(object? value);
}
```

### Built-in validators and composition

**`SkyValidators`** — static factory methods (`Required`, `Email`, `Range`, `MinLength`, …) returning `ISkyValidator`. Each factory creates a small delegate-backed validator.

**`CompositeSkyValidator`** — **What it does:** Runs validators in order, returns first failure. **Why it exists:** Forms need multi-rule validation without a fat `IValidationService`. **OCP role:** Add rules by adding validators to the composite, not by editing a central switch.

```csharp
public sealed class CompositeSkyValidator : ISkyValidator
{
    public SkyValidationResult Validate(object? value)
    {
        foreach (var validator in _validators)
        {
            var result = validator.Validate(value);
            if (!result.IsValid)
                return result;
        }
        return SkyValidationResult.Valid;
    }
}
```

**`SkyFormField`** — binds an input to zero or more `ISkyValidator` instances. Field code calls `Validate` on its composed validator — never individual rule methods.

**Problem → Solution → Walkthrough:**

- **Problem:** `IFormValidationService.ValidateEmailAndRequiredAndRange(...)` grows with every new rule.
- **Solution:** One-method `ISkyValidator`; compose with `CompositeSkyValidator`.
- **Walkthrough:**
  1. ViewModel creates `new CompositeSkyValidator(SkyValidators.Required(), SkyValidators.Email())`.
  2. User edits field; `SkyFormField` reads input value.
  3. Calls `validator.Validate(value)`.
  4. First failing rule returns `SkyValidationResult.Invalid(message)`.
  5. UI displays error; valid input clears error state.

**What would break without this?** Every new validation rule edits a central service. Form fields couple to rules they do not use. Testing email validation requires mocking range, regex, and password-strength methods.

## ISP vs SRP

| Principle | Focus |
|-----------|-------|
| SRP | Class has one reason to change |
| ISP | Interface exposes only what clients need |

A class can satisfy SRP with one public method while its interface still forces unused dependencies on *callers* — that's an ISP problem on the consumer side.

## ISP and DI

In ASP.NET Core / `Microsoft.Extensions.DependencyInjection`, registering `IImageFilterStrategyFactory` separately from `IImageEnhancementStrategyFactory` means handlers request narrow dependencies in constructors — the container resolves only what's needed.

ImgKit's handler registration in `ImgKit.Application/DependencyInjection.cs` wires each handler with its specific factory and gate dependencies — constructor signatures document client roles explicitly.

## Over-Segregation

Too many one-method interfaces create:

- Registration noise
- Unclear discovery ("which of 40 interfaces do I inject?")

**Balance:** segregate when **multiple clients** use **different subsets** of operations. If every client uses every method, one interface is fine.

RainDB's two-level table hierarchy (`ITableSource` → `IColumnarTableSource`) is segregation with a meaningful base — not fragmentation into `ITableName`, `ITableSchema`, `ITableId` as separate injectables.

## Review Checklist

- [ ] Does this interface name describe a single client role?
- [ ] Would a mock implementer need empty stubs for unused methods?
- [ ] Can I split without creating circular dependencies?
- [ ] Do implementers share a meaningful base (e.g., `ITableSource`) for LSP?

## Next

[Dependency Inversion Principle →](06-dependency-inversion.md)
