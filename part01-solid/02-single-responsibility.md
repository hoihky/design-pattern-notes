---
title: Single Responsibility Principle
order: 2
---

# Single Responsibility Principle (SRP)

> *A module should have one, and only one, reason to change.*

— Robert C. Martin

SRP is often misread as "one method per class" or "one public API per file." The correct reading is **one axis of change**: a class should not mix unrelated concerns that would cause different stakeholders or requirements to edit the same code.

## The Core Idea

If changing how markdown is rendered forces you to edit the same class that writes PDF files to disk, that class has **two reasons to change** — rendering policy and export I/O.

Separate them, and each piece can evolve independently. "Reason to change" maps to **stakeholder or requirement axis**: a product manager asking for WeChat support should not force edits in the PDF export module, and a DevOps engineer tuning Puppeteer timeouts should not touch markdown rendering.

### What counts as "one responsibility"?

A responsibility is a **cohesive job** that a module performs for the rest of the system. Ask: *Who would request a change to this code, and why?* If two different people with different goals would both edit the same class, SRP is violated.

Cohesion is the positive mirror of SRP: methods inside a class should cooperate toward one purpose. `RenderMarkdownStep` loops pages and calls a renderer — every line serves "convert markdown to HTML." That is cohesive even though the class has a loop, logging, and cancellation checks.

## Smell: The God Pipeline

Imagine a single `SiteGenerator` class that:

1. Scans the filesystem for `.md` files
2. Parses YAML frontmatter
3. Renders markdown to HTML
4. Rewrites internal links
5. Applies WeChat-specific HTML cleanup
6. Loads Scriban templates
7. Writes output files
8. Exports PDF via headless Chrome

Every new feature — Mermaid support, a new publish mode, footer injection — touches this one class. Tests require spinning up the entire pipeline. Regressions in link rewriting break PDF export tests.

**Why this hurts:** SRP violations compound. The class becomes a **change magnet** — every PR touches it, merge conflicts multiply, and nobody dares refactor because "it might break PDF." Fear of change is a symptom of SRP failure.

## Refactor: One Step, One Class

MDWeb splits generation into discrete **pipeline steps**, each implementing `IGenerationStep`. The interface itself is intentionally minimal — it exists to give the pipeline orchestrator a uniform contract without knowing step internals.

### `IGenerationStep` — the pipeline contract

**What it does:** Defines a single unit of work in site generation. Every step exposes a human-readable `Name` and an `ExecuteAsync` method that mutates a shared `SiteContext`.

**Why it exists:** Without this interface, `SiteGenerationPipeline` would need to know every concrete step type and call them in hard-coded order. The interface lets the orchestrator treat all steps uniformly — classic Chain of Responsibility (Part 4).

**How it fits SRP:** Each implementer owns exactly one transformation. The interface does not bundle "read + render + export" — it bundles only "execute one step."

**Problem it solves:** Decouples *ordering* (pipeline) from *behavior* (individual steps).

```csharp
public interface IGenerationStep
{
    string Name { get; }
    Task ExecuteAsync(SiteContext context, CancellationToken cancellationToken = default);
}
```

### `SiteContext` — shared state, not shared logic

**What it does:** Holds configuration, the content tree, the flat list of pages, theme manifest, and generation outputs. Steps read and write this object; they do not own each other's data structures.

**Why it exists:** Passing ten parameters through every step would violate cohesion in the opposite direction (every step signature would know everything). A context object is the pragmatic compromise — but **logic stays in steps**, not in the context.

### `RenderMarkdownStep` — one transformation only

**What it does:** Iterates `context.AllPages`, calls `IMarkdownRenderer.Render` on each page's raw markdown, and stores HTML plus table-of-contents headings back on the page model.

**Why it exists:** Markdown rendering is a distinct concern from link rewriting, templating, or PDF export. Isolating it means Markdig extension changes touch one file.

**How it fits SRP:** The step depends on `IMarkdownRenderer` (abstraction) rather than Markdig (implementation). Its single reason to change is *how markdown becomes HTML*.

**Problem → Solution → Walkthrough:**

- **Problem:** Rendering logic embedded in a god generator couples HTML output format to unrelated pipeline stages.
- **Solution:** Extract `RenderMarkdownStep` with injected `IMarkdownRenderer`.
- **Walkthrough:**
  1. `SiteGenerationPipeline` invokes steps in DI registration order.
  2. When `RenderMarkdownStep.ExecuteAsync` runs, `context.AllPages` is already populated by `ReadContentStep`.
  3. For each `MarkdownPage`, the step calls `markdownRenderer.Render(page.RawMarkdown)`.
  4. Results are written to `page.HtmlContent` and `page.TableOfContents`.
  5. Later steps (`RewriteLinksStep`, `PostProcessHtmlStep`, `GeneratePagesStep`) consume the HTML — they never re-render markdown.

```csharp
public sealed class RenderMarkdownStep(IMarkdownRenderer markdownRenderer) : IGenerationStep
{
    public string Name => "Render Markdown";

    public Task ExecuteAsync(SiteContext context, CancellationToken cancellationToken = default)
    {
        foreach (var page in context.AllPages)
        {
            var result = markdownRenderer.Render(page.RawMarkdown);
            page.HtmlContent = result.Html;
            page.TableOfContents = result.Headings;
        }
        return Task.CompletedTask;
    }
}
```

**What would break without this?** If rendering lived inside `GeneratePagesStep` (which also applies Scriban templates), a Markdig pipeline change could break page layout tests. If rendering lived inside `ReadContentStep`, content loading would require Markdig in every test that reads files. Separating the step keeps failure domains isolated.

### The full step list and each responsibility

Nine steps register independently in `MDWeb.Application/DependencyInjection.cs`:

| Step | Single responsibility | Reason to change |
|------|----------------------|------------------|
| `ReadContentStep` | Load markdown files into content tree | Content folder layout, frontmatter parsing |
| `NormalizeMarkdownLinksStep` | Normalize link syntax before render | Markdown link conventions |
| `RenderMarkdownStep` | Markdown → HTML | Renderer engine, extensions |
| `RewriteLinksStep` | Fix internal href targets post-render | Output URL scheme |
| `PostProcessHtmlStep` | Apply publish-profile HTML transforms | WeChat, print, AMP rules |
| `LoadThemeStep` | Resolve theme assets and manifest | Theme format |
| `GeneratePagesStep` | Apply Scriban templates to HTML | Template engine, page layout |
| `CopyAssetsStep` | Copy static assets to output | Asset pipeline |
| `ExportPdfStep` | Export composed documents to PDF | Puppeteer, print CSS |

**Reason to change for `RenderMarkdownStep`:** how markdown is converted to HTML (e.g., new Markdig extensions). Not link rewriting, not PDF export.

## Example 2: RainDB — Compile vs Execute

RainDB enforces SRP at the **subsystem** level. A query engine naturally splits into "understand the query" and "run the plan" because those subsystems change for different reasons — SQL dialect features vs execution performance.

### Subsystem boundaries

| Component | Responsibility | Interface | What it must NOT do |
|-----------|----------------|-----------|---------------------|
| SQL compiler | SQL text → logical plan → physical plan | `ISqlCompiler` | Allocate buffers, scan tables |
| LINQ compiler | Expression tree → physical plan | `ILinqCompiler` | Parse SQL strings |
| Query executor | Run a compiled physical plan | `IQueryExecutor` | Parse SQL, bind tables |

The executor's documentation states the boundary explicitly:

```csharp
/// <summary>Runs a compiled physical plan (SRP: parsing/planning live elsewhere).</summary>
public interface IQueryExecutor
{
    ValueTask<IQueryResult> ExecuteAsync(IPhysicalPlan plan, IExecutionContext context);
}
```

**What `IQueryExecutor` does:** Accepts an already-compiled `IPhysicalPlan` and an `IExecutionContext` (catalog, buffer pools, spill writer, cancellation), then dispatches to the appropriate engine (`VectorizedScanEngine`, `HashAggregateEngine`, `JoinExecutionEngine`, …).

**Why it exists:** Execution operators should not re-parse SQL on every query. Compilation produces a plan artifact; execution consumes it. This mirrors real databases (PostgreSQL's parse/plan/execute separation).

**Problem → Solution → Walkthrough:**

- **Problem:** Mixing SQL parsing with hash-join implementation means a lexer bug could destabilize join execution, and vectorized scan optimizations require understanding the parser.
- **Solution:** `DefaultSqlCompiler` produces `IPhysicalPlan`; `DefaultQueryExecutor` runs it.
- **Walkthrough:**
  1. Client sends SQL string to `ISqlCompiler.Compile`.
  2. Compiler binds tables via `ICatalog`, emits a physical plan (e.g., `VectorizedScanPhysicalPlan`).
  3. Client calls `IQueryExecutor.ExecuteAsync(plan, context)`.
  4. Executor pattern-matches plan type, resolves `IColumnarTableSource` from catalog, delegates to specialized engine.
  5. Engine returns `IQueryResult` — executor does not format SQL or re-bind.

**What would break without this?** Changing SQL `GROUP BY` syntax would require editing join execution code. Benchmarking vectorized scans would require constructing SQL strings in tests instead of building plans directly. The compile/execute split lets tests feed hand-crafted `IPhysicalPlan` instances to the executor.

## Example 3: Spark — Editor Panels

Spark's editor divides the UI into **panels**, each implementing `IEditorPanel`. The editor shell (`EditorApplication`) handles docking, layout, and input routing; panels handle domain-specific UI.

### `IEditorPanel` — dock region contract

**What it does:** Defines the lifecycle and identity of one editor region (Hierarchy, Inspector, Project Browser, …). Panels expose a root `IUiElement`, respond to attach/detach/tick events, and can release their widget subtree.

**Why it exists:** Without this interface, `EditorApplication` would embed hierarchy-tree logic, property-grid logic, and asset-browser logic in one class — the C++ equivalent of the god pipeline.

**How it fits SRP:** Each panel implementation owns one editor concern. `HierarchyPanel` manages the scene tree view; `InspectorPanel` shows selected entity properties; `ProjectBrowserPanel` lists project files.

```cpp
class IEditorPanel {
public:
    virtual ~IEditorPanel() = default;
    virtual Utf8String GetPanelId() const = 0;
    virtual Utf8String GetDisplayName() const = 0;
    virtual Ui::IUiElement* GetRootElement() noexcept = 0;
    virtual void OnAttach(EditorContext& ctx) {}
    virtual void OnDetach() {}
    virtual void OnTick(const FrameTiming& timing, EditorContext& ctx) {}
};
```

**Problem → Solution → Walkthrough:**

- **Problem:** A monolithic editor class mixing hierarchy, inspector, and project browser becomes unmaintainable as each panel gains features (search, drag-drop, filtering).
- **Solution:** One class per panel implementing `IEditorPanel`; `EditorApplication` registers and arranges them.
- **Walkthrough:**
  1. Editor startup creates panel instances (`HierarchyPanel`, `InspectorPanel`, …).
  2. `OnAttach` receives `EditorContext` — shared editor state (selection, active scene) without panels reaching into each other's widgets.
  3. Each frame, `OnTick` updates panel-specific logic (e.g., refresh hierarchy when scene changes).
  4. `GetRootElement` returns the panel's widget subtree for the dock shell to layout.
  5. Input events route through the shell; panels handle events on their own elements.

**What would break without this?** Adding inspector undo/redo would risk breaking hierarchy selection. Testing hierarchy filtering would require constructing the entire editor. Panel-level hot reload (rebuild one panel) becomes impossible when everything lives in one translation unit.

## SRP and File/Folder Structure

SRP often correlates with folder layout:

```
MDWeb.Application/Pipeline/
  ReadContentStep.cs
  RenderMarkdownStep.cs
  ExportPdfStep.cs
```

One class per file is not the goal — **one reason to change per class** is. You might legitimately co-locate small helper types with their step if they share the same change axis.

RainDB mirrors this:

```
RainDB.Sql/Compilation/     ← ISqlCompiler implementations
RainDB.Query/Execution/     ← IQueryExecutor, engines
RainDB.Abstractions/        ← interfaces both sides depend on
```

The folder names encode responsibility boundaries. A developer fixing join performance knows to stay in `RainDB.Query/Execution/` without touching SQL parsing.

## Common Misconceptions

| Misconception | Reality |
|---------------|---------|
| "SRP means tiny classes" | A class can be large if it serves one cohesive purpose (e.g., `WeChatHtmlPostProcessor` is long but only changes for WeChat HTML rules) |
| "SRP forbids multiple public methods" | Many methods are fine if they all serve the same responsibility |
| "SRP is only about classes" | Apply to modules, namespaces, and microservices too |
| "Extract until classes are 10 lines" | Over-extraction creates navigation overhead without reducing change coupling |

## SRP vs Other Principles

- **OCP**: SRP makes extension points obvious — small classes are easier to extend via new types than god classes via conditionals
- **ISP**: Often follows from SRP — if a class has one job, its interface stays narrow
- **DIP**: Small classes with single dependencies are easier to invert

## Review Checklist

Ask these questions in code review:

- [ ] Can I describe this class's job in one sentence without "and"?
- [ ] Would two different feature requests edit different parts of this file?
- [ ] Does the class name match its primary behavior?
- [ ] Could I unit-test this class without mocking unrelated subsystems?

## Exercises

1. Open `MDWeb/src/MDWeb.Application/Pipeline/` and list each step's single responsibility in one line.
2. In RainDB, trace a SQL query from `ISqlCompiler` through `IQueryExecutor` — note where compilation ends and execution begins.
3. Find one class in your own codebase that mixes I/O and business logic. Sketch how you would split it.

## Next

[Open/Closed Principle →](03-open-closed.md)
