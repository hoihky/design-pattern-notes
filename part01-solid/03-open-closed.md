---
title: Open/Closed Principle
order: 3
---

# Open/Closed Principle (OCP)

> *Software entities should be open for extension, but closed for modification.*

— Bertrand Meyer

OCP means you add new behavior by **introducing new types** (or registering new plugins) rather than **editing existing conditional logic**. The stable core stays untouched; extensions plug in through abstractions.

## The Core Idea

When the next publish target arrives — WeChat articles, print-only CSS, AMP HTML — you should not open `PostProcessHtmlStep` and add another `if (mode == ...)`. You add a new `IHtmlPostProcessor` implementation and register it in DI.

OCP is about **protecting working code from regression**. Every edit to a conditional branch re-tests all branches. Every new `case` in a `switch` is a merge conflict waiting to happen. Extension via new types isolates change: existing classes compile unchanged; tests for existing processors still pass.

### Extension vs modification

| Approach | What changes | Risk |
|----------|-------------|------|
| Modification | Edit existing method, add branch | Regress existing modes |
| Extension | Add new class, register in DI | Only new code needs tests |

The "closed" part does not mean "never edit." It means the **stable algorithm** — how the step selects and invokes a processor — should not change when a new variant appears.

## Smell: Growing switch Statements

```csharp
// Anti-pattern: every new mode edits this method
string PostProcess(string html, PublishMode mode) => mode switch
{
    PublishMode.Site => html,
    PublishMode.WeChat => WeChatSanitize(html),
    PublishMode.Print => InjectPrintCss(html),  // new requirement
    _ => html
};
```

Each new mode risks regressions in existing modes. Tests must cover every branch on every change. The method becomes a **feature graveyard** — nobody removes dead branches because "something might still use it."

**Why this violates OCP:** The post-processing *algorithm* (iterate pages, transform HTML) is stable, but the *variation axis* (publish mode) keeps forcing edits to the same method.

## Refactor: Strategy Registry

MDWeb defines a narrow extension point. The interface is deliberately small — one property and one method — so new publish profiles implement only what differs.

### `IHtmlPostProcessor` — publish-profile transform

**What it does:** Transforms rendered HTML for a specific `PublishMode`. Each implementation declares which mode it handles via the `Mode` property and implements `Process(string html)`.

**Why it exists:** WeChat Official Account paste workflow requires stripped scripts, inline-only styles, and a restricted tag subset. Standard site generation needs none of that. These are orthogonal transformations on the same HTML artifact.

**How it fits OCP:** Adding print mode means implementing `IHtmlPostProcessor` with `Mode => PublishMode.Print` — not editing `PostProcessHtmlStep`.

**Problem it solves:** Decouples *when* to post-process (pipeline step) from *how* (per-mode strategy).

```csharp
public interface IHtmlPostProcessor
{
    PublishMode Mode { get; }
    string Process(string html);
}
```

### Two implementations — site vs WeChat

**`PassThroughHtmlPostProcessor`** — **What it does:** Returns HTML unchanged. **Why it exists:** Standard site generation needs no post-processing; this explicit no-op makes the registry complete and testable. **Problem it solves:** Callers can inject a real processor in tests without WeChat-specific AngleSharp parsing.

**`WeChatHtmlPostProcessor`** — **What it does:** Parses HTML with AngleSharp, removes unsafe tags (`script`, `iframe`, …), strips `class`/`id`/`data-*` attributes, inlines preset styles from `WeChatStylePresets`, replaces Mermaid blocks with upload placeholders. **Why it exists:** WeChat's editor rejects external stylesheets and JavaScript. **Problem it solves:** Authors write normal markdown; the processor adapts output for paste compatibility.

### `PostProcessHtmlStep` — closed selection logic

**What it does:** Reads `PublishMode` from the theme manifest, finds the matching processor from injected `IEnumerable<IHtmlPostProcessor>`, and applies it to every page's `HtmlContent`.

**Why it stays closed:** The step's control flow — guard check, mode lookup, foreach page — does not change when a third mode arrives. Only DI registration grows.

```csharp
// Simplified from PostProcessHtmlStep.cs
var mode = context.Configuration.ThemeManifest.PublishMode;
var processor = postProcessors.FirstOrDefault(p => p.Mode == mode);
if (processor is null || mode == PublishMode.Site)
    return Task.CompletedTask;

foreach (var page in context.AllPages)
    page.HtmlContent = processor.Process(page.HtmlContent);
```

**Problem → Solution → Walkthrough:**

- **Problem:** WeChat HTML rules diverge sharply from standard site output; inline `if (mode == WeChat)` couples pipeline flow to WeChat-specific AngleSharp logic.
- **Solution:** Extract `IHtmlPostProcessor` implementations; register both in `MDWeb.Infrastructure/DependencyInjection.cs`.
- **Walkthrough:**
  1. `RenderMarkdownStep` produces standard HTML in `page.HtmlContent`.
  2. `PostProcessHtmlStep` runs after rendering and link rewriting.
  3. `SiteGenerationGuard.ShouldGenerateSite` may skip post-processing for export-only runs.
  4. Step resolves processor by `PublishMode` from theme manifest.
  5. For WeChat themes, `WeChatHtmlPostProcessor.Process` sanitizes each page.
  6. Downstream `GeneratePagesStep` receives already-adapted HTML.

**What would break without this?** Every new publish target edits `PostProcessHtmlStep`. WeChat regression tests run on every print-mode change. The step accumulates AngleSharp, print CSS, and AMP logic in one file — an SRP violation stacked on an OCP violation.

## Example 2: Spark — Component Snapshot Handlers

Spark serializes ECS components to scene documents. Each component type (Transform, Mesh, Rigidbody, SpriteAnimator, …) has different capture/restore logic. A single serializer method with a giant switch on component type would violate OCP the moment you add a new component.

### `IComponentSnapshotHandler` — per-component strategy

**What it does:** Captures one component kind from a live `GameObject` into a `ComponentRecord`, and restores it back into a `GameWorld`. Handlers are stateless strategy objects.

**Why it exists:** Component serialization rules vary wildly — a `TransformComponent` stores position/rotation; a `MeshComponent` resolves asset paths via `SceneCaptureContext` callbacks; animation components reference state machines.

**How it fits OCP:** `SceneSerializer` delegates to the registry; new components add a handler file without editing the serializer.

```cpp
class IComponentSnapshotHandler {
public:
    virtual ~IComponentSnapshotHandler() = default;
    virtual ComponentKind GetKind() const noexcept = 0;
    virtual const char* GetKindTag() const noexcept = 0;
    virtual bool TryCapture(const GameObject& owner, const SceneCaptureContext& ctx,
                            ComponentRecord& out) const = 0;
    virtual bool TryRestore(GameObject& owner, const ComponentRecord& record,
                            GameWorld& world, const SceneApplyContext& ctx) const = 0;
};
```

### Supporting types

**`ComponentRecord`** — serialized component payload in the scene document.

**`SceneCaptureContext`** — optional callbacks for asset path resolution during capture (mesh paths, texture keys, skinned model paths).

**`SceneApplyContext`** — hooks during restore (assets root, deferred loading via `GameWorldAssetLoader`, entity-created callbacks).

**`ComponentSnapshotRegistry`** — maps `ComponentKind` → handler. Handlers register at static init or startup via `Register()`.

**Problem → Solution → Walkthrough:**

- **Problem:** `SceneSerializer` with inline switch-on-component-type grows without bound; rendering, physics, and tilemap teams all edit the same file.
- **Solution:** Split handlers across `ComponentSnapshotHandlersRendering.cpp`, `ComponentSnapshotHandlersExtended.cpp`, `ComponentSnapshotHandlersMore.cpp`.
- **Walkthrough:**
  1. Save scene: serializer iterates entities and components.
  2. For each component, registry lookup by `ComponentKind`.
  3. Handler's `TryCapture` writes fields into `ComponentRecord`.
  4. Load scene: for each record, handler's `TryRestore` attaches component to `GameObject`.
  5. Deferred restores (async asset load) use `SceneApplyContext.onDeferredComponent`.

**What would break without this?** Adding `TilemapAutotileComponent` requires editing a 5,000-line serializer. Physics snapshot bugs break rendering tests. Handler logic for unrelated domains interleaves, making code review impossible.

## Example 3: ImgKit — Image Filter Strategies

ImgKit applies filters (blur, sharpen, grayscale, edge enhance, …) through a strategy family. HTTP handlers and LightMediator command handlers depend on factories — never on filter names directly.

### `IImageFilterStrategy` and factory

**What it does:** Each strategy exposes a `Name` and an `Apply(PillowNet.Image, FilterStrategyOptions)` method. The factory builds a case-insensitive dictionary from all registered strategies at startup.

**Why it exists:** Filter algorithms differ in parameters (radius, percent, threshold) and PillowNet API calls. A single handler method with filter-specific branches would grow with every new filter.

**How it fits OCP:** New filter = new sealed strategy class + DI registration. Factory dictionary grows automatically via `IEnumerable<IImageFilterStrategy>` injection.

```csharp
internal sealed class ImageFilterStrategyFactory : IImageFilterStrategyFactory
{
    private readonly IReadOnlyDictionary<string, IImageFilterStrategy> _strategies;

    public ImageFilterStrategyFactory(IEnumerable<IImageFilterStrategy> strategies) =>
        _strategies = strategies.ToDictionary(s => s.Name, StringComparer.OrdinalIgnoreCase);

    public IImageFilterStrategy GetStrategy(string filterName) =>
        _strategies.TryGetValue(filterName, out var strategy)
            ? strategy
            : throw new ArgumentException($"Unsupported filter '{filterName}'.", nameof(filterName));
}
```

ImgKit mirrors this pattern for **`IImageEnhancementStrategyFactory`** (brightness, contrast) and **`IImageOpsStrategyFactory`** (resize, crop, rotate) — three parallel extension axes, each OCP-compliant.

**Problem → Solution → Walkthrough:**

- **Problem:** `ApplyFilterHandler` with inline switch on filter name couples HTTP layer to every PillowNet filter API.
- **Solution:** Handler calls `_filterFactory.GetStrategy(request.FilterName).Apply(image, options)`.
- **Walkthrough:**
  1. API receives filter command with filter name and options.
  2. Handler validates via `ImageOperationValidator`, acquires image through `IPillowNetProcessingGate`.
  3. Factory resolves strategy by name.
  4. Strategy applies PillowNet transform, returns modified image.
  5. Handler writes temp file via `ITempImageFileStore`, returns result DTO.

**What would break without this?** Adding `GaussianBlurFilterStrategy` edits the handler and every test that mocks the handler. Filter-specific validation scatters across one method. Parallel strategy families (enhancement vs ops) would collide in a single god switch.

## OCP Mechanisms in This Book's Projects

| Mechanism | Projects | How extension works |
|-----------|----------|---------------------|
| Interface + DI | MDWeb, ImgKit, LightMediator | New implementation, register in `AddSingleton` |
| Registry | Spark snapshot handlers, collider bake | `Register()` at static init or startup |
| Source generator | LightMapper | New `[LightMapFrom]` pair → generated mapper + DI line |
| Pipeline step list | MDWeb | New `IGenerationStep` in ordered registration |
| Middleware chain | LightMediator | New `IRequestMiddleware<TRequest,TResponse>` |

### LightMapper — OCP via codegen

When you declare a new mapping pair with `[LightMapFrom]`, the source generator emits a concrete mapper class and a line in `AddLightMapperMappers()`. You extend mapping capability without editing runtime mapper infrastructure — OCP enforced at build time.

## OCP vs YAGNI

OCP does not mean "build every extension point on day one." It means when the **second** variant appears, refactor the `if/else` into an abstraction — not when the tenth variant forces a rewrite.

ImgKit started with a few filters; the strategy factory emerged when filter count and test surface grew. MDWeb's `IHtmlPostProcessor` appeared when WeChat mode was real, not when the project started.

**Rule of thumb:** one implementation + no confirmed second variant = inline code is fine. Two implementations = extract the interface.

## When OCP Goes Too Far

Over-abstracting a single implementation creates:

- Interfaces with one implementer forever
- Factories that wrap `new ConcreteType()`
- Plugin registries for two static options

**Rule of thumb:** introduce the abstraction when you have **two** real variants or a **confirmed** upcoming third.

## Review Checklist

- [ ] Can I add a new variant without editing existing classes (only registration/wiring)?
- [ ] Are conditionals on type/mode centralized at the composition root?
- [ ] Do tests for existing variants still pass when I add a new one?
- [ ] Is the extension point named after the **variation axis** (publish mode, filter name, component kind)?

## Connection to Patterns

OCP is the principle behind:

- **Strategy** (Part 4) — interchangeable algorithms
- **Abstract Factory** (Part 2) — interchangeable product families
- **Chain of Responsibility** (Part 4) — new handlers in the chain

## Next

[Liskov Substitution Principle →](04-liskov-substitution.md)
