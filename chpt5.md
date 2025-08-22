# Comprehensive Analysis of Chapter 5: DI Anti-Patterns from "Dependency Injection Principles, Practices, and Patterns"

The latest edition of *Dependency Injection Principles, Practices, and Patterns* by Mark Seemann and Steven van Deursen (published in 2019 by Manning Publications) updates the foundational concepts of Dependency Injection (DI) for modern .NET development, including .NET Core and beyond. This edition emphasizes practical application in loosely coupled systems, with a focus on ASP.NET Core integration. Chapter 5, titled "DI Anti-Patterns," serves as a critical component of Part 2: The DI Catalog. It builds on the patterns introduced in Chapter 4 by highlighting common pitfalls that undermine the benefits of DI, such as reduced maintainability, testability, and modularity. The authors' perspective is clear: anti-patterns are not merely bad habits but systematic errors that invert the Inversion of Control (IoC) principle, leading to tightly coupled code that resists change and hides dependencies.

The chapter's primary objective is to equip developers with the ability to recognize and avoid these anti-patterns early in the design process. Seemann and van Deursen stress that while patterns provide positive guidance, understanding anti-patterns is equally vital for refactoring legacy code and preventing regressions in new projects. They define an anti-pattern as a commonly recurring solution that appears attractive but ultimately generates negative consequences, often violating core DI principles like explicit dependencies and composability.

The authors structure the chapter around four key DI anti-patterns: Control Freak, Bastard Injection, Constrained Construction, and Service Locator. For each, they provide detailed explanations, real-world motivations for why developers fall into them, illustrative examples drawn from an e-commerce application domain (e.g., a Commerce system handling products, repositories, and services), and guidance on why these approaches fail. Where the book's examples are domain-specific, I'll replicate them closely; for broader technical aspects (e.g., implications in .NET Core's service lifetime management), I'll supplement with alternative examples if needed, while staying true to the authors' intent. Every minor yet significant topic is covered, including subtle variations, edge cases, and refactorings to proper DI patterns.

## Introduction to DI Anti-Patterns (Chapter Overview)

Seemann and van Deursen open the chapter by contrasting anti-patterns with the positive DI patterns from Chapter 4 (e.g., Constructor Injection, Method Injection). They argue that anti-patterns often stem from incomplete understanding of IoC: instead of allowing dependencies to be supplied externally, classes seize control, leading to implicit couplings. This violates the Dependency Inversion Principle (DIP), where high-level modules should not depend on low-level ones—both should depend on abstractions.

A key theme is the distinction between *Volatile Dependencies* (those that may change, like database access) and *Stable Dependencies* (e.g., core .NET libraries like `string`). Anti-patterns typically mishandle Volatile Dependencies, making unit testing difficult and deployment brittle. The authors emphasize that recognizing these in code reviews or refactoring sessions can prevent "big ball of mud" architectures. They also note that while some anti-patterns (like Service Locator) were once promoted in early DI literature, evolving practices in .NET Core (e.g., built-in DI container in ASP.NET Core) render them obsolete.

The chapter uses C# code snippets throughout, assuming a .NET Core environment with interfaces for abstractions (e.g., `IProductRepository`). Examples are iterative, building on the Commerce application's `ProductService` class, which calculates discounts and interacts with a repository.

## 1. Control Freak

### Explanation and Author's Perspective
The Control Freak anti-pattern occurs when a class directly instantiates its Volatile Dependencies instead of allowing them to be injected. This inverts IoC by making the class "control" its dependencies, leading to tight coupling. Seemann and van Deursen view this as the most fundamental violation of DI: it hides dependencies, making the class's requirements opaque and unit testing impossible without side effects (e.g., hitting a real database). They argue it's often a symptom of developers new to DI who revert to procedural habits. In .NET Core, this exacerbates issues with service lifetimes (e.g., scoped vs. transient), as the class bypasses the container's management.

Minor topics covered include:
- **Distinction from Stable Dependencies**: Instantiating stable types (e.g., `new List<T>()`) is acceptable, but Volatile ones (e.g., `new SqlConnection()`) are not.
- **Edge Cases**: Factories or builders inside the class can mask Control Freak but still qualify if they create Volatile Dependencies.
- **Motivations for Use**: Convenience in prototypes or when dependencies seem "simple," but this scales poorly.

### Book's Example
The authors illustrate with the `ProductService` class in the Commerce application, which needs an `IProductRepository` to fetch products for discount calculations. In the anti-pattern version:

```csharp
public class ProductService
{
    private readonly IProductRepository repository;

    public ProductService()
    {
        this.repository = new SqlProductRepository();  // Control Freak: Directly instantiating a Volatile Dependency
    }

    public decimal CalculateDiscountedPrice(int productId)
    {
        var product = this.repository.GetById(productId);
        // Apply discount logic...
        return product.Price * 0.9m;  // Simplified discount
    }
}
```

Here, `SqlProductRepository` is a concrete implementation connecting to a SQL database—a Volatile Dependency due to external I/O. The class "freaks out" by not relinquishing control, making it impossible to test `CalculateDiscountedPrice` without a live database.

### Why It's Problematic
- **Testability**: Unit tests can't mock the repository; integration tests are required, slowing development.
- **Maintainability**: Changing to a different repository (e.g., MongoDB) requires modifying `ProductService`, violating Open-Closed Principle.
- **Composability**: The class can't be reused in different contexts (e.g., in-memory for testing).

In .NET Core, this ignores the `IServiceProvider` ecosystem, where dependencies could be resolved with proper scopes.

### Refactoring to Proper DI
The authors recommend refactoring to Constructor Injection (from Chapter 4):

```csharp
public class ProductService
{
    private readonly IProductRepository repository;

    public ProductService(IProductRepository repository)  // Injection point
    {
        this.repository = repository ?? throw new ArgumentNullException(nameof(repository));
    }

    // Same CalculateDiscountedPrice method
}
```

Now, the Composition Root (e.g., in `Startup.cs` for ASP.NET Core) handles instantiation: `services.AddTransient<IProductRepository, SqlProductRepository>();`. This restores IoC.

### Alternative Example (If Book's Doesn't Relate)
For a .NET Core console app not tied to e-commerce, consider a `WeatherService` that fetches data:

Anti-Pattern:
```csharp
public class WeatherService
{
    private readonly HttpClient client = new HttpClient();  // Control Freak
    public async Task<string> GetWeatherAsync(string city) { /* Use client */ }
}
```

Refactored:
```csharp
public class WeatherService
{
    private readonly HttpClient client;
    public WeatherService(HttpClient client) { this.client = client; }
}
```
This leverages .NET Core's `IHttpClientFactory` for better management.

## 2. Bastard Injection

### Explanation and Author's Perspective
Bastard Injection (also called Poor Man's Injection) involves providing a default constructor or overload that instantiates dependencies internally, while optionally allowing injection. The authors see this as a deceptive hybrid: it masquerades as DI-compliant but reintroduces Control Freak for "convenience." It often arises from legacy code transitions or when developers want "easy" instantiation without a container. In .NET Core, this conflicts with explicit registration in `IServiceCollection`, leading to inconsistent lifetimes and hidden defaults that surprise users.

Minor topics:
- **Foreign Defaults**: Defaults from external assemblies amplify coupling.
- **Overload Variations**: Multiple constructors where one injects and another defaults.
- **When It's Acceptable**: Rarely, e.g., for local defaults of Stable Dependencies, but never Volatile ones.

### Book's Example
Using the Commerce app, the anti-pattern appears in `ProductService` with an overloaded constructor:

```csharp
public class ProductService
{
    private readonly IProductRepository repository;

    public ProductService() : this(new SqlProductRepository()) {}  // Bastard Injection: Default to concrete

    public ProductService(IProductRepository repository)
    {
        this.repository = repository ?? throw new ArgumentNullException(nameof(repository));
    }

    // CalculateDiscountedPrice as before
}
```

This allows `new ProductService()` for quick use but defaults to a real SQL repository, making tests hit the database unexpectedly.

### Why It's Problematic
- **Hidden Dependencies**: Callers don't know about the default, leading to runtime surprises (e.g., production config issues).
- **Testability**: Tests using the default constructor couple to the concrete implementation.
- **Maintainability**: Changing the default requires recompiling consumers.

The authors note this anti-pattern erodes trust in the class's API, as constructors should clearly declare requirements.

### Refactoring to Proper DI
Remove the default constructor, forcing injection:

```csharp
public class ProductService
{
    // Only the injecting constructor remains
}
```

For convenience in non-container scenarios, use a factory or explicit creation in the Composition Root.

### Alternative Example
In a .NET Core API controller:

Anti-Pattern:
```csharp
public class ValuesController : ControllerBase
{
    private readonly ILogger logger;

    public ValuesController() : this(new Logger<ValuesController>(new LoggerFactory())) {}  // Bastard

    public ValuesController(ILogger logger) { this.logger = logger; }
}
```

Refactored: Rely on ASP.NET Core's built-in DI to inject `ILogger<ValuesController>` automatically.

## 3. Constrained Construction

### Explanation and Author's Perspective
Constrained Construction happens when a dependency imposes a specific constructor signature on its consumers, forcing awkward adaptations. Seemann and van Deursen argue this anti-pattern stifles composability, as it assumes a fixed creation mechanism rather than abstractions. It's common in third-party libraries without DI-friendly designs. In .NET Core, this clashes with extensible registration patterns, like options builders.

Minor topics:
- **Implicit Constraints**: E.g., requiring parameterless constructors for serialization.
- **Provider Patterns**: Using "providers" that dictate signatures, which the authors dismiss as non-patterns.
- **Edge Cases**: Constraints from frameworks (e.g., WCF proxies).

### Book's Example
In the Commerce app, suppose `SqlProductRepository` requires a connection string in its constructor, but a higher-level service must adapt:

Anti-Pattern (Constrained by dependency):
```csharp
public class SqlProductRepository : IProductRepository
{
    public SqlProductRepository(string connectionString) { /* ... */ }
}

public class ProductService
{
    // Forced to handle connectionString, even if not its concern
    public ProductService(string connectionString)
    {
        this.repository = new SqlProductRepository(connectionString);
    }
}
```

This constrains `ProductService` to know about strings, breaking abstraction layers.

### Why It's Problematic
- **Coupling**: Consumers inherit irrelevant details from dependencies.
- **Flexibility**: Hard to swap implementations without API changes.
- **Testability**: Mocks must mimic the constraint.

### Refactoring to Proper DI
Introduce abstractions or factories:

```csharp
public class ProductService
{
    public ProductService(IProductRepository repository) { /* Injection */ }
}
```

Configure in Composition Root: `services.AddTransient<IProductRepository>(sp => new SqlProductRepository(Configuration.GetConnectionString("Default")));`.

### Alternative Example
For .NET Core configuration: A service constrained by `IConfiguration` directly, instead of injecting options.

Anti-Pattern:
```csharp
public class ConfigService
{
    public ConfigService(IConfiguration config) { /* Constrained to full config */ }
}
```

Refactored: Use `IOptions<MyOptions>` for focused injection.

## 4. Service Locator

### Explanation and Author's Perspective
Service Locator involves a static or passed-in resolver that classes use to fetch dependencies at runtime. The authors strongly condemn this as an anti-pattern that hides dependencies in method bodies, violating explicitness. Seemann famously called it an anti-pattern in his blog, arguing it turns DI into a "magic black box." In .NET Core, the built-in `IServiceProvider` can tempt this, but proper usage is limited to the Composition Root.

Minor topics:
- **Static vs. Injected Locator**: Both hide deps; injected is slightly better but still flawed.
- **Dynamic Resolution**: Useful for plugins, but overused.
- **Comparison to DI Containers**: Locators are containers in disguise, but without composition benefits.

### Book's Example
In Commerce:

```csharp
public class ProductService
{
    private readonly IServiceLocator locator;

    public ProductService(IServiceLocator locator)
    {
        this.locator = locator;
    }

    public decimal CalculateDiscountedPrice(int productId)
    {
        var repository = this.locator.GetInstance<IProductRepository>();  // Service Locator
        var product = repository.GetById(productId);
        // ...
    }
}
```

Dependencies are resolved internally, obscuring what `ProductService` truly needs.

### Why It's Problematic
- **Hidden Dependencies**: No constructor signals requirements; discovery requires code inspection.
- **Testability**: Mocks must configure the locator, complicating setups.
- **Maintainability**: Refactoring breaks if resolutions change; violates SOLID (e.g., Interface Segregation).

The authors note it promotes god classes and hinders parallel development.

### Refactoring to Proper DI
Convert to Constructor Injection:

```csharp
public class ProductService
{
    private readonly IProductRepository repository;

    public ProductService(IProductRepository repository) { this.repository = repository; }
}
```

Reserve `IServiceProvider` for factories or rare dynamic needs in the Root.

### Alternative Example
In ASP.NET Core middleware:

Anti-Pattern:
```csharp
public class MyMiddleware
{
    private readonly RequestDelegate next;

    public MyMiddleware(RequestDelegate next) { this.next = next; }

    public async Task InvokeAsync(HttpContext context)
    {
        var service = context.RequestServices.GetService<IMyService>();  // Locator via HttpContext
        // Use service
    }
}
```

Refactored: Inject via constructor: `public MyMiddleware(RequestDelegate next, IMyService service) { ... }`.

## Chapter Summary and Broader Implications
Seemann and van Deursen conclude by reinforcing that avoiding these anti-patterns leads to cleaner, more testable code. They preview Chapter 6's refactorings, which build on this catalog. From their perspective, DI is a mindset: always favor explicit, injectable dependencies. In .NET Core, leveraging the framework's DI (e.g., `AddScoped`, `AddSingleton`) amplifies these benefits, enabling scalable apps.

Independent research confirms the authors' views: Studies on DI anti-patterns in Java and .NET systems show these patterns correlate with higher bug rates and refactoring costs. For unillustrated concepts like multi-tenancy impacts, consider a tenant-aware repository: Control Freak would hardcode tenant IDs, while proper DI injects a `ITenantContext`. This ensures the chapter's lessons apply broadly.