---
title: Introduction to Structural Patterns
order: 1
---

# Introduction to Structural Patterns

Creational patterns (Part 2) answer **who builds objects**. Behavioral patterns (Part 4) answer **how objects collaborate**. Structural patterns sit between those concerns: they explain how classes and objects are **composed** into larger structures while keeping the design flexible, readable, and efficient.

The recurring structural problems in real codebases are not abstract. They show up as concrete friction:

| Structural problem | What goes wrong without a pattern | Typical fix |
|--------------------|-----------------------------------|-------------|
| **Interface mismatch** | A useful library exposes API A; your client expects API B | Adapter |
| **Abstraction tied to implementation** | Changing rendering backend forces rewriting every panel | Bridge |
| **Part-whole hierarchies** | Client code branches on `if (isFolder)` vs `if (isPage)` everywhere | Composite |
| **Runtime feature stacking** | Subclass explosion for logging + caching + compression | Decorator |
| **Subsystem complexity** | Callers must know nine steps in the right order | Facade |
| **Memory at scale** | Thousands of identical objects each carry duplicate data | Flyweight |
| **Controlled access** | Expensive or sensitive object needs lazy load, gating, or auditing | Proxy |

Structural patterns are about **relationships** — containment, wrapping, delegation, and uniform treatment of tree nodes — not about inventing new algorithms.

## The Seven GoF Structural Patterns

| Pattern | Intent | Core idea |
|---------|--------|-----------|
| **Adapter** | Convert one interface to another clients expect | Wrap an incompatible class; translate calls |
| **Bridge** | Separate abstraction from implementation | Two parallel hierarchies joined by composition |
| **Composite** | Tree structures of part-whole hierarchies | Treat leaf and container nodes through one interface |
| **Decorator** | Add responsibilities dynamically to objects | Stack wrappers that share a common component interface |
| **Facade** | Simplified interface to a subsystem | One entry point delegates to many collaborators |
| **Flyweight** | Share intrinsic state across many objects | Split immutable shared data from per-use context |
| **Proxy** | Surrogate controlling access to another object | Stand-in that forwards (or defers) real work |

## How This Part Is Organized

Each chapter follows the same teaching arc:

1. **Name the structural problem** — interface mismatch, tree composition, subsystem sprawl — before any code appears.
2. **Walk through project examples** — every class gets a role, not just a name.
3. **Trace client usage step by step** — how calling code actually uses the pattern without knowing internals.
4. **Map GoF UML roles to real types** — `Component` becomes `ContentNode`, `Implementor` becomes `IUiBackend`, and so on.

Where a pattern is absent or replaced by a modern idiom (Decorator, Flyweight, Proxy in this corpus), the chapter says so honestly and explains **why** the alternative was chosen.

## Coverage in This Book's Projects

The seven projects in this book (MDWeb, SkyUI, Spark, RainDB, ImgKit, LightMapper, LightMediator) were chosen because they mix domains — static site generation, desktop UI, game engine, embedded database, image API — while sharing C# and C++ idioms. Structural patterns appear at different strengths:

| Pattern | Strength in corpus | Primary examples |
|---------|-------------------|------------------|
| **Adapter** | Excellent | SkyUI `ICheckedListItemAdapter`, MDWeb `MarkdigMarkdownRenderer`, Spark `ManagedGameBridge` |
| **Bridge** | Excellent | Spark `IUiBackend` + `SparkUiBackend` / `DearImguiUiBackend` |
| **Composite** | Excellent | MDWeb `ContentNode` tree, SkyUI `FilterGroupNode`, `CompositeSkyValidator` |
| **Decorator** | Weak — Strategy and pipeline steps used instead | MDWeb `IHtmlPostProcessor`, LightMediator middleware |
| **Facade** | Excellent | MDWeb `SiteGenerator`, Spark `IEngineContext`, RainDB `RainDbEngine` |
| **Flyweight** | Discussed — pooling and singleton palettes vs true Flyweight | SkyUI `SkyDarkColorPalette.Instance`, RainDB `IBufferPool` |
| **Proxy** | Discussed — DI, gates, and `Lazy<T>` replace classic wrappers | ImgKit `IPillowNetProcessingGate`, Spark `TryGet*` accessors |

### Cross-cutting themes

**Composition over inheritance.** Object adapters (`MarkdigMarkdownRenderer` wrapping Markdig), bridge implementors (`DearImguiUiBackend` holding an `IImGuiLayer*`), and composite children lists (`ContentFolder.Children`) all delegate rather than multiply inherit.

**Interfaces as structural glue.** `IMarkdownRenderer`, `ISkyValidator`, `IUiBackend`, and `IVirtualGridDataSource` let clients depend on stable shapes while implementations vary.

**Honest gaps teach too.** Not finding a textbook Decorator chain in production code is informative: when publish modes are mutually exclusive, **Strategy** is clearer than stacking decorators; when concurrency limits matter, a **gate** beats a proxy implementing the full service interface.

## Structural vs Behavioral Boundaries

Some patterns blur category lines:

- **Composite + Visitor** — SkyUI filter trees use Composite for structure and Visitor (`IFilterNodeVisitor<T>`) for operations like SQL export. Composite is Part 3; Visitor is Part 4.
- **Bridge + Abstract Factory** — Spark backends expose `IUiControlsFactory`; each backend ships a matching control family. Bridge holds the relationship; Factory builds the products.
- **Facade vs Mediator** — `SiteGenerator` simplifies subsystem access for the CLI (Facade). `LightMediator.Mediator` routes messages between handlers (Mediator, Part 4).

When in doubt, ask: *Is the pattern mainly about object shape and composition (structural), or about communication and algorithms (behavioral)?*

## Reading Order

Chapters 02–08 each cover one GoF structural pattern in depth. Chapter 08 ends with a summary table and links forward to behavioral patterns, where middleware, visitors, and chains of responsibility extend many of the wrapping and traversal ideas introduced here.

## Next

[Adapter →](02-adapter.md)
