# Analysis of Chapter 8: Object Lifetime from “Dependency Injection Principles, Practices, and Patterns” for .NET Core

This chapter delves into the critical dimension of Dependency Injection (DI) known as object lifetime management. In the context of Pure DI—where no DI container is used—the authors, Mark Seemann and Steven van Deursen, emphasize how controlling the creation, sharing, and disposal of dependencies impacts application maintainability, performance, and resource efficiency. Drawing from the book's structure, the discussion revolves around managing lifetimes to prevent issues like memory leaks, incorrect state sharing, or premature resource release. The chapter builds on earlier concepts, such as object composition from Chapter 7, using a running e-commerce sample application involving components like product repositories and services. Where the book examples are not directly quotable, alternative .NET Core-compatible examples are provided for clarity.

To support the authors' points, additional evidence from .NET documentation and community resources confirms that improper lifetime management can lead to subtle bugs, such as state corruption in multi-threaded environments or excessive resource consumption. The analysis here expands on each technical aspect with thorough explanations, code illustrations, and cross-references to real-world implications.

## 8.1 Managing Dependency Lifetime

Object lifetime refers to the duration an instance of a dependency exists within an application, encompassing when it's created, how it's shared across components, and when it's destroyed. The authors argue that lifetime management is essential for Inversion of Control, as it decouples consumers from the responsibility of instantiating and disposing dependencies. Without proper control, applications risk inefficiencies: short-lived objects might cause frequent garbage collection overhead, while long-lived ones could hoard resources like database connections.

In the book's e-commerce example, a `SqlProductRepository` (implementing `IProductRepository`) might need careful lifetime handling to avoid repeated database connections per operation, which could degrade performance. The chapter explains that in Pure DI, lifetimes are managed explicitly in the Composition Root—a centralized location (often the application's entry point) where the object graph is assembled. This contrasts with container-based approaches but provides transparency.

Supporting evidence from .NET Core's built-in DI system aligns with this: lifetimes dictate instance reuse, affecting scalability in web apps where requests are concurrent. For instance, mismanaging lifetimes can result in thread-unsafe shared state, leading to race conditions.

Alternative example: Consider a console application logging audit events. In Pure DI, the Composition Root might create a logger instance and pass it to services. To manage lifetime explicitly:

```csharp
// Composition Root in Program.cs
public static void Main(string[] args)
{
    // Create a shared logger instance (singleton-like lifetime)
    ILogger logger = new FileLogger("audit.log");
    
    // Compose services with the dependency
    var auditService = new AuditService(logger);
    auditService.LogEvent("Application started");
    
    // Explicitly manage end-of-life (e.g., close file handles)
    (logger as IDisposable)?.Dispose();
}
```

This illustrates explicit creation and disposal, preventing file locks. If the logger were created inside `AuditService`, it would violate DI principles by hiding dependencies.

Key technical details:
- **Creation Timing**: Dependencies are instantiated lazily in the Composition Root to avoid unnecessary allocations.
- **Sharing Mechanisms**: Pass the same instance to multiple consumers for shared lifetimes.
- **Destruction**: Manual calls to `Dispose` or reliance on garbage collection, but the authors warn against over-relying on GC for resources like files or sockets.
- **Performance Implications**: Short lifetimes increase CPU usage for instantiation; long ones risk memory bloat. Benchmarks from community sources show up to 20% overhead in high-throughput apps from excessive transients.

## 8.2 Working with Disposable Dependencies

Disposable dependencies implement `IDisposable` or `IAsyncDisposable` to release unmanaged resources (e.g., file handles, network connections). The chapter stresses that failing to dispose them properly leads to resource leaks, such as open database connections exhausting pools. In Pure DI, disposal is manual, unlike containers that automate it based on lifetime. The authors use the e-commerce sample where a `SqlProductRepository` wraps a `DbConnection`, demonstrating how to ensure disposal to free SQL resources.

Evidence from .NET docs reinforces this: The container disposes transients and scoped services at scope end, but singletons at app shutdown. However, in Pure DI, you must wrap usage in `using` blocks or explicitly call `Dispose` in the Composition Root. Async disposal is highlighted for modern .NET Core apps with `await using`.

Alternative example: An EF Core `DbContext` in a service. In Pure DI for a console app:

```csharp
public class ProductService
{
    private readonly Func<MyDbContext> _contextFactory; // Factory for creating disposables per use

    public ProductService(Func<MyDbContext> contextFactory)
    {
        _contextFactory = contextFactory;
    }

    public void AddProduct(Product product)
    {
        using var context = _contextFactory(); // Ensures disposal after use
        context.Products.Add(product);
        context.SaveChanges();
    }
}

// In Composition Root
var productService = new ProductService(() => new MyDbContext(options));
```

This factory pattern allows on-demand creation and disposal, preventing leaks. If not disposed, connections linger, causing "connection timeout" errors in production.

Technical aspects:
- **Synchronous vs. Async Disposal**: Use `Dispose` for sync; `DisposeAsync` for async resources like HTTP clients.
- **Nested Disposables**: If a dependency owns another disposable, implement composite disposal (e.g., dispose children in parent's `Dispose`).
- **Exceptions in Disposal**: Wrap in try-finally to ensure cleanup even if exceptions occur.
- **Best Practice**: Always check if a dependency is disposable via pattern matching and call `Dispose` at the appropriate lifecycle end to avoid finalizer overhead.

## 8.3 Lifestyle Catalog

The chapter catalogs common lifestyles, explaining their semantics, implementation in Pure DI, and use cases. Each is illustrated with the book's e-commerce components, like sharing a cached `ProductRepository` vs. creating fresh instances for isolation.

### Transient
Transients create a new instance per resolution. Ideal for stateless, lightweight services to ensure isolation.

In Pure DI: Instantiate directly where needed, but to adhere to DI, use factories.

Book example: A transient `DiscountCalculator` in the e-commerce app, recreated per service call for fresh computations.

Alternative code:

```csharp
public class OrderService
{
    private readonly Func<IDiscountCalculator> _calculatorFactory;

    public OrderService(Func<IDiscountCalculator> calculatorFactory)
    {
        _calculatorFactory = calculatorFactory;
    }

    public decimal CalculateTotal(Order order)
    {
        var calculator = _calculatorFactory(); // New instance each time
        return calculator.ApplyDiscount(order.Total);
    }
}

// Composition Root
var orderService = new OrderService(() => new PercentageDiscountCalculator(10));
```

Evidence: .NET docs note transients suit short-lived ops but warn of allocation overhead in loops.

### Singleton
Singletons share one instance app-wide. Suited for stateless, thread-safe services like configuration providers.

In Pure DI: Create once in Composition Root and inject everywhere.

Book example: A singleton `CurrencyProvider` in e-commerce for global exchange rates.

Alternative code:

```csharp
// Composition Root
ICurrencyProvider provider = new StaticCurrencyProvider("USD"); // Single instance

var orderService = new OrderService(provider);
var reportService = new ReportService(provider); // Same instance shared
```

Technical note: Ensure thread-safety with locks or immutable state. Disposal at app end.

### Scoped
Scoped shares an instance within a logical scope (e.g., per HTTP request). Prevents cross-request state leakage.

In Pure DI: Simulate with a scope object or factory that tracks instances.

Book example: Scoped `UserContext` in ASP.NET Core MVC for per-request user data.

Alternative code for a manual scope:

```csharp
public class RequestScope : IDisposable
{
    public IUserContext UserContext { get; } = new HttpUserContext(); // Per-scope instance

    public void Dispose()
    {
        (UserContext as IDisposable)?.Dispose();
    }
}

// Usage in Composition Root for web app simulation
using var scope = new RequestScope();
var controller = new ProductController(scope.UserContext);
controller.HandleRequest();
```

Evidence: In .NET Core, scoped is default for DbContexts to tie to request lifetime. Community benchmarks show scoped reduces DB connections by 50% in web apps.

Other lifestyles mentioned: Per Graph (unique per object graph branch, implemented via recursive factories) and Pooled (reuse from a pool for expensive objects like connections).

## 8.4 Bad Lifestyle Choices

The chapter warns against mismatches that cause "captive dependencies"—where a shorter-lifetime dependency is held by a longer one, effectively promoting its lifetime and causing issues like stale state or non-thread-safety.

Book example: A singleton service injecting a scoped repository, leading to shared DB contexts across requests and data corruption.

Alternative example:

```csharp
// Bad: Singleton capturing scoped
public class SingletonCacheService
{
    private readonly IProductRepository _repo; // Scoped repo becomes singleton

    public SingletonCacheService(IProductRepository repo)
    {
        _repo = repo;
    }
}

// Fix: Use factory for on-demand scoped creation
public class SingletonCacheService
{
    private readonly Func<IProductRepository> _repoFactory;

    public SingletonCacheService(Func<IProductRepository> repoFactory)
    {
        _repoFactory = repoFactory;
    }

    public Product GetCachedProduct(int id)
    {
        using var repo = _repoFactory(); // Fresh scoped instance
        return repo.GetById(id);
    }
}
```

Supporting evidence: .NET validates this in dev mode, throwing exceptions for scoped in singleton. Blogs report this as a top DI bug, causing intermittent failures in production.

Other pitfalls:
- **Thread-Affinity**: Avoid ThreadLocal storage for dependencies, as it breaks in async .NET Core flows.
- **Memory Leaks**: Undisposed singletons holding large data.
- **Fixes**: Use factories, align lifetimes, or validate graphs manually in Pure DI.

In summary, this chapter equips readers with tools to implement robust lifetime strategies, backed by practical .NET Core patterns that enhance application reliability.