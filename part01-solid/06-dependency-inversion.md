---
title: Dependency Inversion Principle
order: 6
---

# Dependency Inversion Principle (DIP)

> *High-level modules should not depend on low-level modules. Both should depend on abstractions. Abstractions should not depend on details.*

— Robert C. Martin

DIP is the principle that makes **dependency injection**, **testability**, and most design patterns possible. Application logic depends on interfaces; infrastructure implements them at the composition root.

## The Core Idea

```
Without DIP:  Application → Markdig, File.ReadAllText, PuppeteerSharp
With DIP:     Application → IMarkdownRenderer, IContentReader, IDocumentExporter
              Infrastructure → concrete implementations
```

The dependency arrow points **inward** toward domain abstractions, not outward toward frameworks.

### Two parts of DIP

1. **High-level modules should not depend on low-level modules** — `RenderMarkdownStep` should not import Markdig types.
2. **Abstractions should not depend on details** — `IMarkdownRenderer` lives in `MDWeb.Core`; it does not reference Markdig's `MarkdownPipeline`.

Both sides depend on the abstraction. Infrastructure "depends on" the interface by implementing it — the interface owns the contract, not the framework.

## Smell: New in Handlers

```csharp
public class RenderMarkdownStep
{
    public void Execute(SiteContext ctx)
    {
        var pipeline = new MarkdownPipelineBuilder().UseAdvancedExtensions().Build();
        foreach (var page in ctx.AllPages)
            page.Html = Markdown.ToHtml(page.RawMarkdown, pipeline);
    }
}
```

Tests must run Markdig. Swapping to a WeChat-specific renderer requires editing the step. The step is **high-level orchestration** coupled to **low-level library details**.

**Why this violates DIP:** The dependency arrow points from application code directly to Markdig — the opposite of inward-pointing abstractions.

## Refactor: Inject Abstractions

MDWeb's `RenderMarkdownStep` receives `IMarkdownRenderer`. Infrastructure registers concrete renderers and selects by publish mode at the composition root.

### `IMarkdownRenderer` — rendering port

**What it does:** Converts markdown string to HTML plus heading metadata. **Why it exists:** Application pipeline should not know whether Markdig or a WeChat-tuned renderer performs conversion. **DIP role:** Application depends on this port; infrastructure adapts Markdig.

### Composition root selection

```csharp
services.AddSingleton<IMarkdownRenderer>(sp =>
    manifest.PublishMode == PublishMode.WeChat
        ? sp.GetRequiredService<WeChatMarkdownRenderer>()
        : sp.GetRequiredService<MarkdigMarkdownRenderer>());
```

**What this wiring does:** Reads theme manifest once at startup, binds the correct renderer implementation. **Why it lives in Infrastructure DI:** Publish-mode selection is a deployment/configuration concern, not a pipeline-step concern.

The application layer never imports Markdig types. Selection logic lives at the composition root (`MDWeb.Infrastructure/DependencyInjection.cs`).

**Problem → Solution → Walkthrough:**

- **Problem:** Pipeline steps instantiate infrastructure directly; tests and alternate deployments cannot swap implementations.
- **Solution:** Define abstractions in Core; register concretions in Infrastructure; inject via constructors.
- **Walkthrough:**
  1. CLI calls `services.AddMDWebApplication()` then `services.AddMDWebInfrastructure(themeDir)`.
  2. Container resolves `SiteGenerationPipeline` with all `IGenerationStep` instances.
  3. `RenderMarkdownStep` receives `IMarkdownRenderer` — container supplies Markdig or WeChat variant.
  4. Step executes without knowing which renderer is active.
  5. Tests register fake renderer: `services.AddSingleton<IMarkdownRenderer, FakeRenderer>()`.

**What would break without this?** Every pipeline test links Markdig. WeChat rendering experiments require editing application code. Application project references Infrastructure assemblies — layering inversion.

## Example 1: MDWeb Abstraction Stack

MDWeb's Core assembly defines the full port surface. Application orchestrates; Infrastructure implements.

| Abstraction | What it abstracts | Implementation | Used by |
|-------------|------------------|----------------|---------|
| `IContentReader` | Loading markdown from content folders | `FileSystemContentReader` | `ReadContentStep` |
| `IMarkdownRenderer` | Markdown → HTML | `MarkdigMarkdownRenderer`, `WeChatMarkdownRenderer` | `RenderMarkdownStep` |
| `IMarkdownLinkRewriter` | Pre-render link normalization | `MarkdownInternalLinkRewriter` | `NormalizeMarkdownLinksStep` |
| `ILinkRewriter` | Post-render href rewriting | `InternalLinkRewriter` | `RewriteLinksStep` |
| `IHtmlPostProcessor` | Publish-profile HTML transforms | `PassThroughHtmlPostProcessor`, `WeChatHtmlPostProcessor` | `PostProcessHtmlStep` |
| `ITemplateEngine` | HTML → final page | `ScribanTemplateEngine` | `GeneratePagesStep` |
| `IOutputWriter` | Writing files to output directory | `FileSystemOutputWriter` | `GeneratePagesStep`, `CopyAssetsStep` |
| `IDocumentExporter` | PDF generation | `PuppeteerPdfDocumentExporter` | `ExportPdfStep` |
| `IThemeManifestLoader` | Theme configuration loading | `FileSystemThemeManifestLoader` | Composition root |

**`SiteGenerator`** orchestrates `IEnumerable<IGenerationStep>` — it does not know filesystem or Markdig details.

**Data flow with DIP intact:**

1. `ReadContentStep` → `IContentReader.ReadAsync` → populates `SiteContext`
2. `RenderMarkdownStep` → `IMarkdownRenderer.Render` → fills `HtmlContent`
3. `PostProcessHtmlStep` → `IHtmlPostProcessor.Process` → adapts HTML per mode
4. `GeneratePagesStep` → `ITemplateEngine.Render` + `IOutputWriter.Write`
5. `ExportPdfStep` → `IDocumentExporter.Export` → PDF on disk

Each arrow crosses an abstraction boundary. Swapping Markdig for another engine touches Infrastructure registration only.

## Example 2: Spark — IEngineContext

Game and demo code depend on a stable facade, not the monolithic `Engine` class.

### `IEngineContext` — engine facade for game code

**What it does:** Exposes window, frame presenter, input, framebuffer size, scene render params, optional sound engine, scene pointer, and ImGui layer through virtual methods.

**Why it exists:** Demos and games should not include Vulkan, GLFW, or audio backend headers. The engine implementation changes; game code compiles against a narrow stable surface.

**How it fits DIP:** High-level game logic depends on `IEngineContext`; `EngineContext` (in engine internals) implements and forwards.

```cpp
class IEngineContext {
public:
    virtual ~IEngineContext() = default;
    virtual Window& GetWindow() = 0;
    virtual IFramePresenter& GetFramePresenter() = 0;
    virtual IInput& GetInput() = 0;
    virtual void GetFramebufferSize(int& outWidth, int& outHeight) const = 0;
    virtual SoundEngine* TryGetSoundEngine() noexcept = 0;
    virtual Scene* TryGetScene() noexcept = 0;
    virtual IImGuiLayer* TryGetImGuiLayer() noexcept = 0;
};
```

**Supporting abstractions:**

- **`IFramePresenter`** — abstracts Vulkan/OpenGL present path
- **`IInput`** — keyboard/mouse/gamepad without GLFW types in game headers
- **`IUiBackend`** + **`IUiControlsFactory`** — UI toolkit without Dear ImGui in consumer code

**Problem → Solution → Walkthrough:**

- **Problem:** Demo code including `Engine.hpp` pulls in rendering, audio, ECS, and platform headers.
- **Solution:** Pass `IEngineContext&` to demo `OnInit`/`OnTick`; engine bootstrap creates `EngineContext`.
- **Walkthrough:**
  1. Engine constructs subsystems (window, presenter, input, sound).
  2. `EngineContext` implements `IEngineContext`, holds references.
  3. Demo receives context in lifecycle hooks.
  4. Demo calls `ctx.GetInput()` for movement, `ctx.TryGetScene()` for ECS access.
  5. Rendering params set via `SetSceneRenderParams` without Vulkan types.

**What would break without this?** Headless test builds would link graphics drivers. Scripting bindings would expose engine internals instead of the facade. Platform ports require rewriting every demo.

## Example 3: LightMapper + ImgKit

LightMapper inverts mapping from hand-written property copies to generated, injectable mappers.

### `ILightMapper<TSource, TDestination>` — mapping port

**What it does:** `Map(source)` creates and fills a destination; `MapTo(source, destination)` updates an existing instance. **Why it exists:** Handlers and services should not contain repetitive DTO mapping code. **DIP role:** Application depends on generic mapper interface; source generator produces implementations.

```csharp
public interface ILightMapper<TSource, TDestination>
    where TDestination : notnull
{
    TDestination Map(TSource source);
    void MapTo(TSource source, TDestination destination);
}
```

**Source generator** emits concrete mapper classes (e.g., for `[LightMapFrom(typeof(TargetDto))]` on a source type) and `AddLightMapperMappers()` extension that registers each as `ILightMapper<TSource,TDestination>` singleton.

ImgKit consumes mapping without referencing generated type names:

```csharp
services.AddLightMapperMappers();
```

Handlers depend on `ILightMapper<SourceDto, TargetModel>` — inversion from inline mapping to generated, testable mappers.

**Problem → Solution → Walkthrough:**

- **Problem:** `CommandModelMapper` static helpers couple handlers to every DTO shape; new commands edit central mapping code.
- **Solution:** Declare mapping pairs with attributes; generator emits mappers; DI registers abstractions.
- **Walkthrough:**
  1. Developer adds `[LightMapFrom(typeof(ResizeImageSpecification))]` on command type.
  2. Build runs incremental generator → emits mapper class + DI registration line.
  3. Handler injects `ILightMapper<ResizeImageCommand, ResizeImageSpecification>`.
  4. Handler calls `_mapper.Map(request)` instead of manual property copy.
  5. Tests substitute fake mapper or test generated mapper in isolation.

**What would break without this?** Every new API command edits shared mapping utilities. Handlers reference generated concrete types — compile coupling to generator output names. Unit tests cannot replace mapping without rewriting handlers.

## Example 4: RainDB — IExecutionContext

`DefaultQueryExecutor` depends on a bundle of abstractions passed per query — not on concrete allocators or file paths.

### `IExecutionContext` — per-query resource bundle

**What it does:** Provides `ICatalog`, `IBufferPool`, `IAlignedBufferPool`, `ISpillWriter`, and `CancellationToken` to execution engines. **Why it exists:** Operators (scan, join, aggregate) need shared resources but should not locate them globally. **DIP + OCP role:** Add tracing or metrics to context without changing operator signatures.

```csharp
public interface IExecutionContext
{
    ICatalog Catalog { get; }
    IBufferPool BufferPool { get; }
    IAlignedBufferPool AlignedBufferPool { get; }
    ISpillWriter SpillWriter { get; }
    CancellationToken CancellationToken { get; }
}
```

**`RainDbExecutionContext`** — concrete implementation created per query in `RainDbEngine`.

Operators never `new` concrete allocators. Tests substitute `NoOpSpillWriter.Instance` and in-memory catalogs via a test context.

**Problem → Solution → Walkthrough:**

- **Problem:** Engines call `ArrayPool.Shared` and static catalog — untestable, non-configurable.
- **Solution:** Pass `IExecutionContext` into every `ExecuteAsync` call.
- **Walkthrough:**
  1. Client compiles SQL → `IPhysicalPlan`.
  2. Engine creates `RainDbExecutionContext` with catalog, pools, spill writer.
  3. `DefaultQueryExecutor.ExecuteAsync(plan, context)` dispatches to engine.
  4. `VectorizedScanEngine` allocates from `context.AlignedBufferPool`.
  5. Large join may spill via `context.SpillWriter`; tests use no-op spill.

**What would break without this?** Executor tests would write temp files on disk. SIMD benchmarks could not use custom pool implementations. Cancellation would not propagate into operators.

## DIP and the Composition Root

DIP only works if **something** wires concretions. That place is the composition root:

| Project | Composition root | What gets wired |
|---------|------------------|-----------------|
| MDWeb | `MDWeb.Cli/Program.cs`, `DependencyInjection.cs` | Renderers, readers, steps, exporters |
| ImgKit | `ImgKit.Web/Program.cs`, `ImgKit.Application/DependencyInjection.cs` | Strategies, handlers, mapper registrations |
| RainDB | `RainDbEngine.CreateDefault()` | Catalog, compiler, executor, buffer pools |
| Spark | Engine bootstrap / editor startup | Context facade, UI backend, snapshot handlers |
| LightMediator | `LightMediatorServiceCollectionExtensions` | Mediator, publisher, handler discovery |
| LightMapper | Generated `AddLightMapperMappers()` | All `ILightMapper<,>` singletons |

Business logic classes should not call `new` on infrastructure types. The composition root is the only place that chooses Markdig over WeChat, sequential over parallel publisher, or filesystem over in-memory catalog.

## DIP vs DI Framework

- **DIP** is a design principle (depend on abstractions)
- **DI** is a technique (constructor injection, service locator, factory)
- **IoC container** is a tool (`IServiceCollection`, manual wiring)

You can follow DIP without a container — pass interfaces via constructors and construct the graph in `Main`. Spark does this with manual registries and constructor injection in C++.

## Review Checklist

- [ ] Do high-level modules import only abstractions from Core/Abstractions assemblies?
- [ ] Is framework code (Markdig, PillowNet, Vulkan) confined to Infrastructure?
- [ ] Can I unit-test with fakes without filesystem or network?
- [ ] Does the composition root contain all `new` calls for services?

## Next

[SOLID in Practice →](07-solid-in-practice.md)
