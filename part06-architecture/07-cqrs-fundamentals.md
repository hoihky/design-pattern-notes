---
title: CQRS Fundamentals
order: 7
---

# CQRS Fundamentals

**Command Query Responsibility Segregation (CQRS)** separates **operations that change state** (commands) from **operations that read state** (queries). The split can be at the application layer (separate handlers), the data layer (separate read/write stores), or both.

Greg Young popularized application-level CQRS; the idea applies anywhere reads and writes have different scaling, consistency, or modeling needs.

## The Problem CQRS Solves

A single service method that both reads and writes accumulates complexity:

```csharp
public async Task<OrderDto> ProcessOrderAndGetSummary(OrderRequest req)
{
    // validate, save, send email, update cache, return summary...
}
```

Problems:

- **Read and write concerns entangled** — changing email policy risks breaking read mapping
- **Different scaling** — reads dominate traffic; writes need transactions
- **Testing** — must mock entire stack to test a simple lookup
- **API clarity** — POST that silently returns a dashboard aggregate confuses clients

CQRS says: **one message type, one handler, one responsibility.**

## Commands vs Queries

| | Command | Query |
|---|---------|-------|
| **Intent** | Change system state | Return data |
| **Side effects** | Expected | Should be none (ideally) |
| **Naming** | `CreateOrderCommand`, `ResizeImageCommand` | `GetOrderSummaryQuery`, `GetImageInfoQuery` |
| **Return** | Result DTO or `Unit` (void) | Read model / DTO |
| **HTTP mapping** | POST, PUT, DELETE | GET (or POST for complex reads) |

**Strict CQRS** uses separate models for read and write (different DTOs, sometimes different databases). **CQRS-lite** (common in this book) shares a store but separates **handlers** and **message types**.

## LightMediator Message Taxonomy

**LightMediator** implements application CQRS via three message kinds:

### 1. IRequest&lt;TResponse&gt; — Command or Query with result

```csharp
public interface IRequest<out TResponse> { }
```

- `CreateOrderCommand : IRequest<CreateOrderResult>` — **command** (writes)
- `GetOrderSummaryQuery : IRequest<GetOrderSummaryResponse>` — **query** (reads)

Naming convention distinguishes intent; the interface is the same.

### 2. IRequest — Command with no result

```csharp
public interface IRequest : IRequest<Unit> { }
```

`DeleteOrderCommand : IRequest` — delete succeeds or throws; no payload returned.

### 3. INotification — Side-effect fan-out

```csharp
public interface INotification { }
```

`OrderCreatedNotification` — multiple handlers (logging, analytics) without bloating `CreateOrderHandler`.

**Send vs Publish:**

| Method | Cardinality | Use |
|--------|-------------|-----|
| `SendAsync(command)` | Exactly one handler | Command or query |
| `PublishAsync(notification)` | Zero or many handlers | Domain events, side effects |

## Handler Contract

```csharp
public interface IRequestHandler<in TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    Task<TResponse> HandleAsync(TRequest request, CancellationToken cancellationToken);
}
```

**One handler per request type** — enforced at registration. Contrast with notifications (many handlers).

Handler responsibilities:

1. Validate request (or delegate to pipeline middleware)
2. Execute use case (call ports, not HTTP/UI)
3. Return response DTO

Handler should **not** send emails, log analytics, and update cache inline — publish notifications instead.

## Mediator — Dispatch Hub

```csharp
public interface IMediator
{
    Task<TResponse> SendAsync<TResponse>(IRequest<TResponse> request, CancellationToken ct);
    Task PublishAsync<TNotification>(TNotification notification, CancellationToken ct)
        where TNotification : INotification;
}
```

Controllers and ViewModels depend on `IMediator` — not on ten handler interfaces. **Mediator pattern** (Part 4) enables **CQRS** message routing.

## Middleware Pipeline

```csharp
public interface IRequestMiddleware<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    Task<TResponse> InvokeAsync(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct);
}
```

Cross-cutting concerns wrap handlers onion-style:

- Logging elapsed time (`ElapsedRequestLoggingMiddleware`)
- Validation, caching, authorization

Business handlers stay focused on one use case.

## Read vs Write Models

### Shared store (LightMediator sample)

`IOrderStore` serves both `CreateOrderHandler` (write) and `GetOrderSummaryHandler` (read). Handlers are separate; storage is shared. Good for learning handler separation before event sourcing.

### Layered models (ImgKit)

```
HTTP Request DTO
  → Command/Query (contract)
    → Specification (domain, via LightMapper)
      → ProcessedImageModel (internal)
        → ProcessedImageResult (contract response)
          → API Response DTO
```

Each layer has a distinct shape — queries skip mutation specifications.

## CQRS Is NOT...

| Confused with | Difference |
|---------------|------------|
| **UI `ICommand`** (MVVM) | Button binding vs application message |
| **GoF Command pattern** | CQRS uses Command *concept* at app boundary; often `record` types + handlers |
| **Pipeline** (`IGenerationStep`) | Sequential shared context vs isolated request/response |
| **CRUD repository** | CQRS splits read/write *operations*, not just table access |

SkyUI's `FilterEditorRelayCommand` binds UI — not CQRS. MDWeb's pipeline mutates shared `SiteContext` — not CQRS.

## When to Adopt CQRS

| Signal | Consider CQRS |
|--------|-------------|
| Handlers growing with unrelated read/write logic | Yes |
| Read models differ greatly from write models | Yes |
| Need independent read scaling | Yes (possibly separate read DB) |
| Simple CRUD with 5 endpoints | Probably overkill |
| Stateless transform API (ImgKit) | Commands dominate; few queries — still valuable |

## Next

[CQRS with LightMediator and ImgKit →](08-cqrs-practice.md)
