---
title: Introduction to SOLID
order: 1
---

# Introduction to SOLID

SOLID is not a design pattern catalog — it is a set of five **design principles** that guide how you structure classes, interfaces, and dependencies so that software remains understandable, testable, and open to change.

Robert C. Martin (Uncle Bob) popularized the acronym:

| Letter | Principle | One-line summary |
|--------|-----------|------------------|
| **S** | Single Responsibility | One reason to change per module |
| **O** | Open/Closed | Extend behavior without editing existing code |
| **L** | Liskov Substitution | Subtypes must honor the base contract |
| **I** | Interface Segregation | Small, focused interfaces |
| **D** | Dependency Inversion | Depend on abstractions, not concretions |

## What SOLID Actually Is

Each letter names a **constraint on coupling**. Coupling is not inherently bad — every line of code that calls another line creates a dependency — but *accidental* coupling is what makes systems brittle. When a change to WeChat HTML sanitization forces you to re-test PDF export, that is accidental coupling between unrelated concerns. SOLID gives you vocabulary and heuristics to spot and reduce that coupling before it calcifies.

The principles are not independent. **Single Responsibility** makes **Open/Closed** practical (small classes are easier to extend). **Dependency Inversion** makes **Liskov Substitution** testable (you can inject fakes). **Interface Segregation** prevents fat abstractions that violate **Liskov** when half the methods throw `NotSupportedException`. Treat SOLID as a system, not a checklist of five unrelated rules.

## Why SOLID Matters Before Patterns

Design patterns solve **recurring structural and behavioral problems**. SOLID tells you **when** a pattern is appropriate and **how** to shape the abstractions patterns rely on.

Consider MDWeb's site generation pipeline. It uses the **Chain of Responsibility** pattern (Part 4), but each step stays small because of **Single Responsibility**. New publish modes (WeChat) arrive via **Open/Closed** — a new `IHtmlPostProcessor` instead of editing the pipeline. Steps receive `IMarkdownRenderer` through **Dependency Inversion**, not Markdig directly.

Without SOLID, patterns become ceremony: factories for one class, strategies with a single implementation, facades that hide nothing. A `Strategy` pattern where the strategy interface has twelve methods and one implementation is not a strategy — it is a renamed god class.

### Problem → Solution → Walkthrough: MDWeb Pipeline

**Problem:** Static site generation touches filesystem I/O, YAML frontmatter parsing, markdown rendering, link rewriting, HTML post-processing for different publish targets, Scriban templating, asset copying, and headless-Chrome PDF export. Putting all of that in one `SiteGenerator` class means every feature request collides in the same file.

**Solution:** Split the pipeline into discrete `IGenerationStep` implementations. Each step owns one transformation on the shared `SiteContext`. `SiteGenerationPipeline` orchestrates steps in order without knowing their internals.

**Walkthrough — control flow:**

1. **Composition root** (`MDWeb.Infrastructure/DependencyInjection.cs`) registers concrete services (`FileSystemContentReader`, `MarkdigMarkdownRenderer`, …) against abstractions (`IContentReader`, `IMarkdownRenderer`, …).
2. **Application layer** (`MDWeb.Application/DependencyInjection.cs`) registers nine `IGenerationStep` implementations in pipeline order.
3. **CLI or watch service** builds a `SiteContext` from configuration and calls `SiteGenerationPipeline.ExecuteAsync`.
4. **Each step** reads from and writes to `SiteContext` — for example, `ReadContentStep` populates `context.AllPages`, then `RenderMarkdownStep` fills each page's `HtmlContent`.
5. **Infrastructure never imports Application** — dependency arrows point inward toward `MDWeb.Core.Abstractions`.

**What would break without this?** A monolithic generator would require full filesystem + Markdig + Puppeteer for every unit test. Adding WeChat mode would mean editing a 2,000-line method and re-running the entire integration suite. Merge conflicts on the god class would become daily.

## SOLID vs Patterns — A Mental Model

```mermaid
flowchart LR
    SOLID[SOLID Principles] -->|shape| ABSTR[Abstractions]
    ABSTR -->|enable| PATTERNS[Design Patterns]
    PATTERNS -->|compose into| SYSTEMS[Real Systems]
```

- **SOLID** answers: "Is this class doing too much? Can I swap implementations? Will subclasses break callers?"
- **Patterns** answer: "How do I structure creation, composition, and collaboration?"

Think of SOLID as **grammar** and patterns as **vocabulary**. You need grammar first — otherwise every sentence is a run-on.

## The Seven Projects as a Teaching Corpus

This book uses seven projects that share a common architectural style:

1. **Interface-first design** — abstractions live in core/assemblies; infrastructure implements them
2. **Dependency injection** — composition roots wire concrete types at startup
3. **Explicit pattern comments** — many types are labeled `// Builder pattern` or `// Abstract Factory` in source

| Project | Primary SOLID strengths | Key abstractions to study |
|---------|------------------------|---------------------------|
| MDWeb | SRP pipeline steps, OCP post-processors, DIP throughout | `IGenerationStep`, `IHtmlPostProcessor`, `IMarkdownRenderer` |
| RainDB | SRP compile vs execute, ISP layered table interfaces | `ISqlCompiler`, `IQueryExecutor`, `ITableSource`, `IColumnarTableSource` |
| Spark | ISP UI controls, LSP physics shapes, DIP engine facade | `IShape2D`, `IEditorPanel`, `IEngineContext`, `IComponentSnapshotHandler` |
| LightMediator | SRP per-handler, LSP notification publishers | `IMediator`, `IRequestHandler<T,R>`, `INotificationPublisher` |
| LightMapper | DIP generated mapper registrations | `ILightMapper<TSource,TDestination>` |
| ImgKit | OCP strategy families, ISP separate factories | `IImageFilterStrategyFactory`, `IImageEnhancementStrategyFactory` |
| SkyUI | ISP validators, OCP composable validation | `ISkyValidator`, `CompositeSkyValidator` |

### How the projects relate architecturally

All seven follow a similar **onion**:

```
┌─────────────────────────────────────┐
│  Host (CLI, API, Demo, Editor)      │  ← composition root
├─────────────────────────────────────┤
│  Application / orchestration      │  ← use cases, handlers, pipeline
├─────────────────────────────────────┤
│  Core / Abstractions                │  ← interfaces, models, contracts
├─────────────────────────────────────┤
│  Infrastructure / engine internals  │  ← Markdig, PillowNet, Vulkan, …
└─────────────────────────────────────┘
```

MDWeb's `MDWeb.Core` and RainDB's `RainDB.Abstractions` sit at the center. Spark achieves the same layering in C++ via header directories (`include/spark/engine/` vs `src/spark/`). The principle is identical: **high-level code never includes low-level headers unless through a narrow interface**.

## What "Practical" Means in This Book

We avoid textbook `Animal` / `Dog` hierarchies. Every example:

- Comes from **shipping code** you can find and study at `https://hoihky.github.io`
- Shows **trade-offs** (when a principle is stretched, or when DI lifetime replaces Singleton)
- Connects to **patterns** covered in later parts

When we cite a class, we explain **what it does**, **why it exists**, and **what breaks if you remove the abstraction**. Class names alone are not documentation.

## How to Read Part 1

Each principle chapter follows the same structure:

1. **Definition** — precise wording and common misconceptions
2. **Smell** — code that violates the principle
3. **Refactor** — how the principle guides the fix
4. **Project examples** — two or three real excerpts with Problem → Solution → Walkthrough
5. **What would break without this?** — concrete failure modes
6. **Checklist** — questions to ask during code review

After all five principles, Chapter 7 surveys how SOLID appears across all seven projects and lists frequent pitfalls (god classes, leaky abstractions, over-segregation).

### Suggested reading pace

Read each chapter with the corresponding project open in your editor. For Chapter 2 (SRP), open `MDWeb/src/MDWeb.Application/Pipeline/` and read one step file per section. For Chapter 4 (LSP), run Spark's physics tests in `spark/tests/physics/ShapeNarrowPhaseTest.cpp` after reading the shape interface. Active reading — tracing call paths — beats passive skimming.

## A Note on C# and C++

C# examples lean on interfaces, `IServiceCollection`, and `async`/`await`. C++ examples (Spark) use pure virtual interfaces, `UniquePtr`, and explicit registries. The principles are language-agnostic; the idioms differ.

| Concept | C# idiom | C++ idiom (Spark) |
|---------|----------|-------------------|
| Abstraction | `interface IMarkdownRenderer` | `class IShape2D { virtual … = 0; }` |
| Composition root | `DependencyInjection.cs`, `Program.cs` | Engine bootstrap, static `Register()` |
| Extension without edit | `services.AddSingleton<IHtmlPostProcessor, …>()` | `ComponentSnapshotRegistry::Register(handler)` |
| Test double | Mock via DI container | `NoOpSpillWriter.Instance`, in-memory catalog |

## What Would Break Without SOLID? (Cross-Cutting)

Across all seven projects, removing SOLID discipline produces predictable symptoms:

| Symptom | Which principle was ignored | Example |
|---------|----------------------------|---------|
| God class with 40 methods | SRP | One `SiteGenerator` doing everything |
| `switch (mode)` grows every sprint | OCP | Inline publish-mode logic in `PostProcessHtmlStep` |
| `is CircleShape` checks everywhere | LSP | Shape hierarchy with partial implementations |
| Mocks stub 15 unused methods | ISP | Fat `IImageService` for a resize endpoint |
| Tests require real filesystem | DIP | `new FileSystemContentReader()` inside steps |

These are not theoretical — they are the regressions the reference projects were structured to prevent.

## Next

[Single Responsibility Principle →](02-single-responsibility.md)
