# Chapter 4: DI Patterns

In this chapter of "Dependency Injection Principles, Practices, and Patterns" (second edition, updated for .NET Core), the authors, Mark Seemann and Steven van Deursen, delve into the core patterns that enable Dependency Injection (DI). The focus is on practical ways to implement DI in your code to achieve loose coupling. The chapter assumes you've grasped the basics from earlier chapters, like why DI matters and how tightly coupled code contrasts with loosely coupled designs.

The authors emphasize that DI isn't just about using a container; it's a set of patterns for composing objects. They introduce four key patterns: Composition Root, Constructor Injection, Method Injection, and Property Injection. Each is explained with its mechanics, appropriate use cases, known applications, and real-world examples drawn from an e-commerce application scenario. The goal is to show how these patterns help manage dependencies without hard-coding them, making code more testable, maintainable, and extensible.

The authors' perspective is pragmatic: Start with Pure DI (manual wiring without a container) to understand the fundamentals, then layer on containers where scale demands it. They warn against overcomplicating simple apps and stress that patterns should solve real problems, not be applied blindly.

## 4.1 Composition Root

The Composition Root is the foundational pattern for DI. It's the single location in your application where you assemble all dependencies and compose the object graph. This keeps the rest of the codebase clean and unaware of how dependencies are created or managed.

### 4.1.1 How Composition Root Works

The Composition Root acts as the entry point for dependency composition. In a typical application, it's placed as close as possible to the main entry point (e.g., in a console app's `Main` method or an ASP.NET Core app's `Startup` class). Here, you create instances of dependencies and inject them into higher-level objects.

Key principles:
- **Only one per application**: Multiple roots lead to confusion and bugs.
- **Close to the entry point**: Minimizes the distance dependencies travel.
- **Uses other DI patterns**: Relies on Constructor Injection, etc., to wire things up.
- **Supports testing**: You can compose different graphs for unit tests.

Technically, in .NET Core, this often involves the `IServiceCollection` in ASP.NET Core, but the authors advocate starting with Pure DI to avoid container magic hiding issues.

### 4.1.2 Using a DI Container in a Composition Root

While Pure DI is ideal for learning, containers like Autofac, Simple Injector, or Microsoft.Extensions.DependencyInjection simplify large graphs. The container is configured in the Composition Root, registering types and resolving the root object.

Example workflow:
1. Register abstractions and implementations.
2. Resolve the root service.
3. Let the container handle transitive dependencies.

The authors note: Containers don't change the need for a Composition Root; they just automate parts of it. Misuse can lead to anti-patterns (covered in Chapter 5).

### 4.1.3 Example: Implementing a Composition Root using Pure DI

The book uses an e-commerce sample where a `HomeController` needs a `ProductService` to display featured products. Without DI, `HomeController` would new up `ProductService` directly—tight coupling.

With Pure DI in the Composition Root (e.g., in `Program.cs` for a console-like setup or `Startup.ConfigureServices` in ASP.NET Core):

```csharp
// Abstractions
public interface IProductRepository { IEnumerable<Product> GetFeaturedProducts(); }
public interface IProductService { IEnumerable<Product> GetFeaturedProducts(); }

// Implementations
public class SqlProductRepository : IProductRepository
{
    public IEnumerable<Product> GetFeaturedProducts() { /* SQL query */ return new List<Product>(); }
}

public class ProductService : IProductService
{
    private readonly IProductRepository _repository;

    public ProductService(IProductRepository repository)
    {
        _repository = repository ?? throw new ArgumentNullException(nameof(repository));
    }

    public IEnumerable<Product> GetFeaturedProducts()
    {
        return _repository.GetFeaturedProducts();
    }
}

// Composition Root (e.g., in Program.Main or Startup)
var repository = new SqlProductRepository();
var service = new ProductService(repository);
var controller = new HomeController(service);  // HomeController takes IProductService in constructor
// Use controller...
```

This composes the graph manually. The authors highlight guard clauses (e.g., null checks) to fail fast on missing dependencies.

### 4.1.4 The Apparent Dependency Explosion

As apps grow, the Composition Root might seem bloated with many `new` statements. This "explosion" is illusory—it's just making implicit dependencies explicit. It's a sign of good design: All wiring is centralized, and changes are isolated here.

The authors advise: If it gets unwieldy, introduce facades or use a container, but don't distribute composition logic.

## 4.2 Constructor Injection

Constructor Injection is the most common DI pattern, where dependencies are passed via the constructor. It's the authors' default choice for mandatory dependencies.

### 4.2.1 How Constructor Injection Works

The class declares its dependencies as constructor parameters (preferably abstractions). The caller (usually the Composition Root or container) supplies them.

Benefits:
- Ensures the object is always fully initialized.
- Makes dependencies immutable (via readonly fields).
- Supports immutable objects, aligning with functional principles.

In C#:
```csharp
public class MyClass
{
    private readonly IDependency _dep;

    public MyClass(IDependency dep)
    {
        _dep = dep ?? throw new ArgumentNullException(nameof(dep));
    }
}
```

### 4.2.2 When to Use Constructor Injection

Use for:
- Required dependencies.
- When the dependency is used throughout the class's lifetime.
- Stateless services.

Avoid if the dependency is optional or changes per method call (use Method Injection then).

### 4.2.3 Known Use of Constructor Injection

Common in frameworks like ASP.NET Core (controllers inject services via constructors). Also in domain services, repositories, etc.

### 4.2.4 Example: Adding Currency Conversions to the Featured Products

In the e-commerce app, featured products need prices converted to the user's currency. Introduce an `ICurrencyConverter` dependency into `ProductService`.

Abstractions:
```csharp
public interface ICurrencyConverter
{
    decimal Convert(decimal amount, string fromCurrency, string toCurrency);
}
```

Updated `ProductService`:
```csharp
public class ProductService : IProductService
{
    private readonly IProductRepository _repository;
    private readonly ICurrencyConverter _converter;

    public ProductService(IProductRepository repository, ICurrencyConverter converter)
    {
        _repository = repository ?? throw new ArgumentNullException(nameof(repository));
        _converter = converter ?? throw new ArgumentNullException(nameof(converter));
    }

    public IEnumerable<Product> GetFeaturedProducts(string userCurrency)
    {
        var products = _repository.GetFeaturedProducts();
        return products.Select(p => new Product
        {
            Name = p.Name,
            Price = _converter.Convert(p.Price, p.Currency, userCurrency)
        });
    }
}
```

Composition Root update:
```csharp
var repository = new SqlProductRepository();
var converter = new ExchangeRateConverter();  // Implements ICurrencyConverter, perhaps using an API
var service = new ProductService(repository, converter);
```

This keeps `ProductService` decoupled from conversion logic. The authors explain: This enables testing with mocks and swapping converters without changing `ProductService`.

### 4.2.5 Wrap-up

Constructor Injection enforces honest APIs—callers see exactly what's needed. It prevents partial initialization and aligns with SOLID principles (especially Dependency Inversion).

## 4.3 Method Injection

Method Injection passes dependencies as parameters to specific methods, ideal for dependencies needed only occasionally.

### 4.3.1 How Method Injection Works

Dependencies are method arguments, not stored as fields. This avoids holding state for rarely used deps.

In C#:
```csharp
public void DoSomething(IDependency dep)
{
    // Use dep here
}
```

### 4.3.2 When to Use Method Injection

- For dependencies varying per call.
- When the dependency introduces temporal coupling (e.g., must be called in sequence).
- For cross-cutting concerns like logging in specific methods.

Not for class-wide deps—use Constructor Injection.

### 4.3.3 Known Use of Method Injection

In domain events (pass event args), or in factories where the created object needs a temp dep.

### 4.3.4 Example: Adding Currency Conversions to the ProductEntity

`Product` is a simple entity, but needs conversion without always holding a converter (to keep it lightweight).

```csharp
public class Product
{
    public string Name { get; set; }
    public decimal Price { get; set; }
    public string Currency { get; set; }

    public decimal ConvertPrice(string toCurrency, ICurrencyConverter converter)
    {
        if (converter == null) throw new ArgumentNullException(nameof(converter));
        return converter.Convert(Price, Currency, toCurrency);
    }
}
```

Usage:
```csharp
var product = new Product { Price = 100, Currency = "USD" };
var converter = new ExchangeRateConverter();
var convertedPrice = product.ConvertPrice("EUR", converter);
```

This injects the converter only when needed. The authors note: This avoids bloating the entity with unused fields and allows flexible converter choice per call.

## 4.4 Property Injection

Property Injection sets dependencies via public setters, for optional or replaceable deps.

### 4.4.1 How Property Injection Works

Expose a writable property (abstraction). Caller sets it after creation.

In C#:
```csharp
public class MyClass
{
    public IDependency Dependency { get; set; } = new DefaultDependency();  // Optional default
}
```

### 4.4.2 When to Use Property Injection

- Optional dependencies with sensible defaults.
- For extensibility in libraries (users can override).
- Cyclic dependencies (rare, but possible).

Avoid for required deps—use Constructor Injection to enforce.

### 4.4.3 Known Uses of Property Injection

In frameworks for plugins (e.g., set a custom logger). ASP.NET Core uses it for optional services.

### 4.4.4 Example: Property Injection as an Extensibility Model of a Reusable Library

Imagine a reusable logging library with a default console sink, but allowing custom sinks.

```csharp
public interface ILogSink
{
    void Write(string message);
}

public class ConsoleLogSink : ILogSink
{
    public void Write(string message) => Console.WriteLine(message);
}

public class Logger
{
    public ILogSink Sink { get; set; } = new ConsoleLogSink();

    public void Log(string message)
    {
        Sink.Write(message);
    }
}
```

Usage:
```csharp
var logger = new Logger();
// Default: console

// Customize:
logger.Sink = new FileLogSink("log.txt");
logger.Log("Hello");
```

The authors explain: This makes the library extensible without forcing users to provide a sink. In an app, the Composition Root can set it.

## 4.5 Choosing Which Pattern to Use

The authors provide guidance to select patterns based on context. Here's a summary table for quick reference:

| Pattern              | Best For                          | Pros                              | Cons                              | Example Scenario                  |
|----------------------|-----------------------------------|-----------------------------------|-----------------------------------|-----------------------------------|
| Composition Root    | Application wiring              | Centralizes composition; Easy to change | Can seem bloated in large apps   | Wiring entire e-commerce services |
| Constructor Injection | Required, class-wide deps        | Ensures initialization; Immutable | Increases param count             | Injecting repository into service |
| Method Injection   | Per-call, variable deps          | Flexible; No state overhead       | Less discoverable                 | Passing converter to entity method|
| Property Injection | Optional, extensible deps         | Allows defaults; Runtime swap     | Can mutate state; Less explicit   | Customizing library components    |

Prioritize Constructor Injection for reliability. Use Method/Property for flexibility. Combine as needed (e.g., Constructor for core deps, Property for optional). Always consider testability: Can I mock this easily?

The chapter reinforces the authors': DI patterns are enablers for loosely coupled code. Master them in Pure DI first, use examples like the e-commerce to practice, and you'll see how they make complex systems manageable without magic. This sets the stage for anti-patterns in Chapter 5, where misuse is dissected.