---
title: Clean Architecture in Practice
order: 6
---

# Clean Architecture in Practice

This chapter walks four projects that implement the dependency rule differently — from textbook four-layer MDWeb to contracts-first RainDB and header-based Spark.

---

## MDWeb — Textbook Four-Layer Split

MDWeb is the clearest Clean Architecture example in the corpus.

### Project Map

```
MDWeb.Core/              ← innermost: zero project references
  Abstractions/          ← 15+ port interfaces
  Models/                ← SiteConfiguration, ContentNode, MarkdownPage
MDWeb.Application/       ← use cases only → references Core
  Pipeline/              ← IGenerationStep implementations
  Generation/            ← SiteGenerator facade
  Building/              ← SiteBuilder
MDWeb.Infrastructure/    ← adapters → references Core only
  Rendering/             ← MarkdigMarkdownRenderer
  Templating/            ← ScribanTemplateEngine
  Content/               ← FileSystemContentReader
MDWeb.Cli/               ← composition root → Application + Infrastructure
```

### Dependency Graph

```
MDWeb.Cli
  ├── MDWeb.Application ──→ MDWeb.Core
  └── MDWeb.Infrastructure ──→ MDWeb.Core
```

Application and Infrastructure are **siblings**. Infrastructure never references Application.

### Port → Adapter Table

| Port (`MDWeb.Core`) | Adapter (`MDWeb.Infrastructure`) | What it hides |
|---------------------|----------------------------------|---------------|
| `IContentReader` | `FileSystemContentReader` | Filesystem scan, frontmatter parse |
| `IMarkdownRenderer` | `MarkdigMarkdownRenderer`, `WeChatMarkdownRenderer` | Markdig pipeline config |
| `ITemplateEngine` | `ScribanTemplateEngine` | Scriban script execution |
| `IOutputWriter` | `FileSystemOutputWriter` | Directory create, file write |
| `IDocumentExporter` | `PuppeteerPdfDocumentExporter` | Headless Chrome PDF |
| `IHtmlPostProcessor` | `PassThroughHtmlPostProcessor`, `WeChatHtmlPostProcessor` | Publish-mode HTML transforms |

### Use Case Example — RenderMarkdownStep

Application layer step depends on port:

```csharp
public sealed class RenderMarkdownStep(IMarkdownRenderer markdownRenderer) : IGenerationStep
{
    public Task ExecuteAsync(SiteContext context, CancellationToken ct)
    {
        foreach (var page in context.AllPages)
        {
            var result = markdownRenderer.Render(page.RawMarkdown);
            page.HtmlContent = result.Html;
        }
        return Task.CompletedTask;
    }
}
```

**No `using Markdig`** in Application. Swapping renderers is Infrastructure + composition root concern.

### Composition Root — Program.cs

```csharp
services.AddMDWebApplication();
services.AddMDWebInfrastructure(themeDir);
```

Infrastructure DI selects renderer by theme manifest:

```csharp
services.AddSingleton<IMarkdownRenderer>(sp =>
    manifest.PublishMode == PublishMode.WeChat
        ? sp.GetRequiredService<WeChatMarkdownRenderer>()
        : sp.GetRequiredService<MarkdigMarkdownRenderer>());
```

**Walkthrough:** CLI parses `--theme` → loads manifest → registers WeChat or site renderer → pipeline steps receive correct strategy without conditional logic in steps.

---

## ImgKit — Application-Centric Hexagon

ImgKit uses a pragmatic layout: ports live in `ImgKit.Application/Abstractions`, not a separate Core project.

### Layer Map

```
ImgKit.Api/           ← REST controllers (presentation)
ImgKit.Web/           ← Blazor studio (presentation)
ImgKit.Application/   ← use cases, handlers, strategies, abstractions
ImgKit.Infrastructure/← thin: PillowNet runtime bootstrap
```

### Dependency Direction

```
Api → Application, Infrastructure
Application ↛ Api, Web (no upward refs)
Handlers → ITempImageFileStore, IImageFilterStrategyFactory (ports in Application)
```

### Thin Controller — Fat Handler

`ImagesController` maps HTTP to commands:

```csharp
[HttpPost("resize")]
public async Task<IActionResult> Resize(IFormFile image, [FromForm] ResizeImageRequest request, ...)
{
    var command = new ResizeImageCommand { Image = await image.ToImageInputAsync(...), ... };
    var result = await mediator.SendAsync(command, cancellationToken);
    return ToFileResult(result);
}
```

Controller knows HTTP (`IFormFile`). Handler knows image processing. **Separation of presentation from use case.**

### Cross-Library Ports

ImgKit consumes **LightMediator** (dispatch) and **LightMapper** (DTO mapping) as application-layer infrastructure — handlers stay free of manual property copying and direct handler resolution.

### Pragmatic Leak

`TempImageFileStore` concrete class registers inside Application DI — file I/O slightly inside the "use case" ring. Acceptable for small apps; larger systems move it to Infrastructure.

---

## RainDB — Contracts-First Engine

RainDB optimizes for **library consumers** embedding an analytics engine.

### Assembly Rings

```
RainDB.Abstractions/   ← ICatalog, IQueryExecutor, IPhysicalPlan, IBufferPool
RainDB.Core/           ← catalog impl, MemoryTable, columnar storage
RainDB.Query/          ← execution engines, physical operators
RainDB.Sql/            ← SQL text → physical plan
RainDB.Linq/           ← expression trees → physical plan
RainDB.Driver/           ← RainDbEngine composition root + public API
```

### SRP at Subsystem Boundary

| Compiler | Responsibility |
|----------|----------------|
| `ISqlCompiler` | SQL → `IPhysicalPlan` |
| `ILinqCompiler` | Expression → `IPhysicalPlan` |
| `IQueryExecutor` | Run plan, return `IQueryResult` |

`IQueryExecutor` XML docs: *"SRP: parsing/planning live elsewhere."*

### RainDbEngine — Facade + Composition Root

```csharp
public sealed class RainDbEngine
{
    public ICatalog Catalog { get; }
    public ISqlCompiler SqlCompiler { get; }
    public ILinqCompiler LinqCompiler { get; }
    public IQueryExecutor Executor { get; }

    public static RainDbEngine CreateDefault(ICatalog catalog)
    {
        var buffers = new HybridBufferPool();
        return new RainDbEngine(catalog, buffers, buffers,
            new DefaultQueryExecutor(), new DefaultSqlCompiler(), new DefaultLinqCompiler(),
            NoOpSpillWriter.Instance);
    }
}
```

Consumers call `engine.ExecuteSqlAsync("SELECT ...")` — not `new DefaultSqlCompiler()` manually.

### IExecutionContext — DIP Bundle

Operators receive execution dependencies as one port:

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

Tests inject in-memory catalog and `NoOpSpillWriter.Instance` without touching operator code.

---

## Spark — Clean Architecture in C++

Spark uses **headers as public contracts** instead of .NET assemblies.

### Layout

```
include/spark/     ← public API (IEngineContext, IGame, IShape2D, IUiBackend)
src/spark/         ← Vulkan, GLFW, implementations (games must not include)
```

### Dependency Inversion for Games

Games implement `IGame` and receive `IEngineContext`:

```cpp
class IEngineContext {
    virtual Window& GetWindow() = 0;
    virtual IInput& GetInput() = 0;
    virtual Scene* TryGetScene() noexcept = 0;
};
```

Demo code never `#include`s Vulkan headers. `Engine` owns subsystems; `EngineContext` implements the facade.

### Documented Philosophy

Spark's programming guide states: **games depend on `IGame` / `IEngineContext`, not `VulkanRenderer`.**

Same dependency rule, native code idioms.

---

## SkyUI — Package Layering (Not Domain CA)

SkyUI is a **UI framework**, not a business application. Layering is horizontal:

```
SkyUI.Core → SkyUI → SkyUI.Data / SkyUI.Diagram
```

`ISkyColorPalette`, `IVirtualGridDataSource`, `IEdgePathComputer` are **framework extension ports** — OCP for themes and routing, not enterprise domain.

Still teaches **modular presentation architecture**.

---

## Comparison Table

| Project | Inner ring | Composition root | Best teaching angle |
|---------|-----------|------------------|---------------------|
| MDWeb | `MDWeb.Core` | `MDWeb.Cli/Program.cs` | Strict four-layer CA |
| ImgKit | Application abstractions | `ImgKit.Api/Program.cs` | CQRS + hexagonal pragmatism |
| RainDB | `RainDB.Abstractions` | `RainDbEngine.CreateDefault` | Engine/library CA |
| Spark | `include/spark/` | `main.cpp` / `Engine` ctor | C++ DIP |
| SkyUI | `SkyUI.Core` | Demo `App.axaml` | UI package modularity |

## Next

[CQRS Fundamentals →](07-cqrs-fundamentals.md)
