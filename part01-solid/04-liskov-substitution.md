---
title: Liskov Substitution Principle
order: 4
---

# Liskov Substitution Principle (LSP)

> *Subtypes must be substitutable for their base types without altering the correctness of the program.*

— Barbara Liskov

LSP is about **behavioral contracts**. If code accepts `Base`, any `Derived` must honor what callers expect: preconditions, postconditions, invariants, and error semantics.

## The Core Idea

Polymorphism is only safe when substituting `B` for `A` never surprises the caller. Violations often appear as:

- Overridden methods that throw `NotSupportedException`
- Subtypes that tighten preconditions ("caller must pass non-null X, but base allowed null")
- Subtypes that weaken postconditions (base promises sorted results; derived returns arbitrary order)

LSP is the principle that makes `interface` and inheritance trustworthy. Without it, every call site needs `is` checks or defensive coding — polymorphism becomes a lie.

### Behavioral subtyping vs structural subtyping

Structural subtyping (common in TypeScript) says "if it has the methods, it fits." LSP adds: **the methods must behave compatibly**. A type that compiles but throws on half its methods is structurally valid and behaviorally broken.

## Smell: The Square/Rectangle Problem

Classic teaching example: `Square` inherits `Rectangle` and overrides `setWidth` to also set height. Code that expects independent width/height breaks when given a `Square`.

Real codebases rarely use geometry — but the same failure mode appears with "special" implementations that don't fully support the base interface. The .NET framework's `ReadOnlyCollection<T>` wrapping `IList<T>` and throwing on `Add` is the canonical C# example.

## Example 1: Spark — Physics Shapes

Spark's 2D collision system accepts `IShape2D&` everywhere — narrow phase, broad phase, raycasts, contact generation. Every concrete shape must implement the full contract.

### `IShape2D` — collision geometry contract

**What it does:** Defines the complete set of geometric queries a 2D physics shape must answer: bounding AABB, overlap tests against other shapes/AABBs/circles, raycasts, and contact manifold computation.

**Why it exists:** Collision detection code (`NarrowPhase2D`, `Collider2D`, tilemap baking) should not know whether it is testing a circle or a convex polygon. Polymorphism replaces type switches.

**How it fits LSP:** Every concrete shape — `CircleShape2D`, `BoxShape2D`, `ConvexPolygonShape2D` — provides real implementations for all methods. No shape returns "unsupported" for core queries.

**Problem it solves:** Uniform collision pipeline without `if (type == Circle)` scattered through physics code.

```cpp
class IShape2D {
public:
    virtual ~IShape2D() = default;
    virtual ShapeType2D GetType() const noexcept = 0;
    virtual CollisionAabb2 GetBounds() const noexcept = 0;
    virtual bool Overlaps(const IShape2D& other) const = 0;
    virtual bool OverlapsAabb(const CollisionAabb2& aabb) const = 0;
    virtual bool OverlapsCircle(float centerX, float centerY, float radius) const = 0;
    virtual bool Raycast(const Ray2D& ray, float& outDistance) const = 0;
    virtual bool ComputeContact(const IShape2D& other, ContactManifold2D& out) const = 0;
};
```

### Concrete shapes and the narrow phase

**`CircleShape2D`** — analytic circle-circle and circle-AABB tests.

**`BoxShape2D`** — axis-aligned or oriented rectangle tests via SAT (separating axis theorem).

**`ConvexPolygonShape2D`** — general convex polygon contact via `ShapeContact2DDetail`.

**`NarrowPhase2D`** — dispatches on shape-type pairs (circle-box, polygon-polygon, …) without null checks or "unsupported" fallbacks.

**Problem → Solution → Walkthrough:**

- **Problem:** Physics code with partial shape support requires runtime type checks and error paths that complicate the hot loop.
- **Solution:** Full interface implementation per shape; factory (`ShapeFactory2D`) creates `UniquePtr<IShape2D>`.
- **Walkthrough:**
  1. Collider holds `UniquePtr<IShape2D>`.
  2. Broad phase collects overlapping pairs using `GetBounds()`.
  3. Narrow phase calls `Overlaps(other)` or `ComputeContact(other, manifold)`.
  4. Raycast queries use `Raycast(ray, outDistance)` — on miss, distance is unchanged.
  5. Contact resolution consumes `ContactManifold2D` regardless of shape types.

**LSP test:** pass any `IShape2D` to `Overlaps`, `Raycast`, and `ComputeContact`; behavior is shape-appropriate but never violates the interface's implicit guarantees (e.g., raycast returns false with `outDistance` unchanged when no hit).

**What would break without this?** A `MeshShape2D` that throws on `ComputeContact` would crash the physics step when broad phase pairs it with a circle. Callers would need `dynamic_cast` guards — polymorphism defeated. Tests in `spark/tests/physics/ShapeNarrowPhaseTest.cpp` assume all shapes are fully substitutable.

## Example 2: RainDB — Query Result Hierarchy

Query results form a small hierarchy rooted at `IQueryResult`. Callers that only need row counts work with any subtype; callers that need columnar batches opt into the richer interface.

### Interface layering

**`IQueryResult`** — **What it does:** Exposes `RowCount` and async disposal. **Why it exists:** Every query returns something countable and disposable, whether columnar scan or aggregate. **LSP role:** Base contract all result types honor.

**`IColumnarQueryResult`** — **What it does:** Adds `IReadOnlyList<IColumnarBatch> Batches`. **Why it exists:** Scan and join results expose columnar data for vectorized consumers. **LSP role:** Extends base without breaking callers that only read `RowCount`.

**`IAggregateQueryResult`** — **What it does:** Aggregate-specific result surface. **LSP role:** Same pattern — additive capability.

```csharp
public interface IQueryResult : IAsyncDisposable
{
    long RowCount { get; }
}

public interface IColumnarQueryResult : IQueryResult
{
    IReadOnlyList<IColumnarBatch> Batches { get; }
}
```

**`EmptyQueryResult`** — internal implementation returning zero rows with no-op dispose. Honors the base contract: valid `RowCount`, safe `DisposeAsync`.

**Problem → Solution → Walkthrough:**

- **Problem:** Callers assume all results expose batches; aggregate-only results force null checks or exceptions.
- **Solution:** Layer interfaces; callers cast or pattern-match only when they need columnar data.
- **Walkthrough:**
  1. `DefaultQueryExecutor.ExecuteAsync` returns `IQueryResult`.
  2. Logging/metrics code reads `RowCount` without knowing result shape.
  3. Client needing batches checks `result is IColumnarQueryResult columnar` and iterates `Batches`.
  4. `await result.DisposeAsync()` releases buffers regardless of subtype.

**Violation to avoid:** an `EmptyQueryResult` that throws on `Dispose` or reports negative `RowCount` would break substitutability. RainDB's implementation returns `RowCount` from constructor and completes dispose synchronously.

**What would break without this?** Forcing all results into one fat type with nullable `Batches` would violate ISP and tempt callers to ignore null semantics. Throwing on unsupported properties would violate LSP.

## Example 3: LightMediator — Notification Publishers

LightMediator routes domain notifications to handlers. The fan-out strategy — sequential vs parallel — is swappable via `INotificationPublisher`.

### `INotificationPublisher` — fan-out contract

**What it does:** Dispatches a notification to all registered `INotificationHandler<TNotification>` instances.

**Why it exists:** Some domains require ordered side effects (audit log before email). Others benefit from parallel I/O (send push + update cache concurrently). The mediator should not hard-code either policy.

**How it fits LSP:** Both publishers implement the same method signature. Callers invoke `PublishAsync`; handlers run. The contract is "all handlers are invoked" — not "handlers run in registration order" unless sequential publisher is registered.

```csharp
public interface INotificationPublisher
{
    Task PublishAsync<TNotification>(
        TNotification notification,
        IServiceProvider serviceProvider,
        CancellationToken cancellationToken)
        where TNotification : INotification;
}
```

### Sequential vs parallel implementations

**`SequentialNotificationPublisher`** — resolves handlers from DI, awaits each `HandleAsync` in registration order. **Use when:** handlers have ordering dependencies or share non-thread-safe state.

**`ParallelNotificationPublisher`** — resolves handlers, launches all `HandleAsync` calls, awaits `Task.WhenAll`. **Use when:** handlers are independent and thread-safe.

**`Mediator`** — depends on `INotificationPublisher` via constructor injection. `PublishAsync` delegates entirely to the publisher; the mediator never branches on publisher type.

```csharp
public sealed class SequentialNotificationPublisher : INotificationPublisher
{
    public async Task PublishAsync<TNotification>(...)
    {
        var handlers = serviceProvider.GetServices<INotificationHandler<TNotification>>();
        foreach (var handler in handlers)
            await handler.HandleAsync(notification, cancellationToken);
    }
}
```

**Problem → Solution → Walkthrough:**

- **Problem:** Hard-coded sequential dispatch prevents parallel notification handling without editing `Mediator`.
- **Solution:** Extract `INotificationPublisher`; register preferred implementation in DI.
- **Walkthrough:**
  1. Application calls `mediator.PublishAsync(new OrderCreated(...))`.
  2. `Mediator` forwards to `_notificationPublisher.PublishAsync`.
  3. Publisher resolves all `INotificationHandler<OrderCreated>` from `IServiceProvider`.
  4. Sequential: handlers run one-by-one. Parallel: all start, `Task.WhenAll` waits.
  5. Exceptions propagate according to publisher policy (sequential stops at first failure; parallel aggregates).

**Document the difference:** LSP allows different *performance* characteristics, but callers must not assume sequential ordering when parallel publisher is registered. The contract is "handlers are invoked," not "handlers run in registration order" unless documented and sequential publisher is wired.

**What would break without this?** A parallel publisher that silently drops handlers on exception would violate expectations set by sequential behavior. Tests using one publisher would not predict production behavior with the other — an LSP documentation gap, not a code bug.

## LSP and Exceptions

A subtype that throws where the base never threw violates LSP if callers aren't prepared. Example anti-pattern:

```csharp
class ReadOnlyList : IList<T>
{
    public void Add(T item) => throw new NotSupportedException();
}
```

`IList<T>` implies mutability to many callers. Prefer `IReadOnlyList<T>` at the abstraction boundary (ISP + LSP).

RainDB avoids this with interface layering: `ITableSource` for metadata-only consumers, `IColumnarTableSource` for scan engines. Metadata clients never see batch APIs they cannot use.

## LSP in Testing

Substitutability enables **test doubles**:

- `NoOpSpillWriter.Instance` (RainDB) substitutes for `ISpillWriter` in unit tests — writes nothing, honors dispose semantics
- `PassThroughHtmlPostProcessor` substitutes for real post-processing in pipeline tests — returns input unchanged
- In-memory `ICatalog` substitutes for file-backed catalog in executor tests

If fakes violate invariants the production code assumes, tests pass but production fails — an LSP smell in the test design. A fake `ISpillWriter` that throws on every write would not substitute for production spill behavior.

## Review Checklist

- [ ] Does every override honor the base method's documented contract?
- [ ] Can I pass any implementation to code written against the interface without `is` checks?
- [ ] Do subclasses avoid throwing `NotSupportedException` on core operations?
- [ ] Are stronger preconditions (stricter validation) documented so callers know?

## Next

[Interface Segregation Principle →](05-interface-segregation.md)
