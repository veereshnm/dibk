### Chapter 3: Writing Loosely Coupled Code

This chapter builds directly on the foundation laid in Chapter 2, where the authors presented a tightly coupled implementation of an e-commerce application for a fictional developer named Mary Rowan. The goal here is to refactor that application into a loosely coupled version using Dependency Injection (DI) principles. The authors emphasize that understanding bad practices (from Chapter 2) is crucial for appreciating good ones, much like knowing why you should let a grilled steak rest before slicing it—to redistribute the juices for a juicier result. Cutting too soon lets the juices run out, making the meat dry and less enjoyable. Similarly, tightly coupled code leads to brittle, hard-to-maintain systems, while loosely coupled code allows for better redistribution of responsibilities, making the application more flexible, testable, and extensible.

The chapter is structured around rebuilding Mary's e-commerce application with DI, analyzing the new implementation, and evaluating its benefits. The authors use the same domain: an ASP.NET Core web application that displays featured products with user-specific discounts and currency conversions. The volatile dependencies (like data access and user context) are inverted using abstractions and injected via constructors. All technical details, including code examples, are preserved as in the book, with explanations of why each change promotes loose coupling. The authors introduce key DI patterns: Constructor Injection (the primary method used), Method Injection (for occasional use), and the Composition Root (where the object graph is assembled).

#### 3.1 Rebuilding the E-Commerce Application

The authors start by refactoring the tightly coupled code from Chapter 2 step by step. In Chapter 2, the application had hard-coded dependencies: the `HomeController` directly instantiated `ProductService`, which in turn instantiated `SqlProductRepository` and relied on `HttpContext` for user information. This led to issues like poor testability, rigidity, and violated principles like the Single Responsibility Principle (SRP) and Open-Closed Principle (OCP).

Here, they introduce abstractions for volatile dependencies (e.g., data access and user context) and use Constructor Injection to pass them in. This inverts control, allowing higher-level components to depend on abstractions rather than concrete implementations.

##### 3.1.1 Building a More Maintainable UI

The UI layer is an ASP.NET Core MVC controller. In the tightly coupled version, `HomeController` created its own `ProductService` instance, mixing construction logic with business logic. This violates SRP and makes testing difficult (e.g., you can't mock the service without changing the controller).

To fix this, the authors introduce Constructor Injection in `HomeController`. They define no new interface for `ProductService` yet (as it's a stable domain component), but inject it directly:

```csharp
using Microsoft.AspNetCore.Mvc;
using Commerce.Domain; // Namespace for domain model

public class HomeController : Controller
{
    private readonly ProductService productService;

    public HomeController(ProductService productService)
    {
        this.productService = productService ?? throw new ArgumentNullException(nameof(productService));
    }

    public ViewResult Index()
    {
        var products = this.productService.GetFeaturedProducts();
        return this.View(products);
    }
}
```

Key technical aspects:
- **Guard clause**: The constructor uses `?? throw new ArgumentNullException` to ensure the dependency is provided, preventing null references (a defensive programming technique emphasized by the authors).
- **Loose coupling benefit**: The controller no longer knows how to create `ProductService`. This makes it easier to swap implementations or mock for unit tests.
- **No interface for ProductService yet**: The authors note that interfaces are only needed for volatile dependencies. `ProductService` is stable, so direct injection is fine for now.

The view (Index.cshtml) remains the same, rendering the list of products with prices and discounts.

##### 3.1.2 Building an Independent Domain Model

The domain layer (`ProductService`) was tightly coupled to concrete data access (`SqlProductRepository`) and user context (`HttpContext`). The authors refactor it to depend on abstractions: `IProductRepository` for data access and `IUserContext` for user information.

First, define the abstractions:

```csharp
public interface IProductRepository
{
    IEnumerable<Product> GetFeaturedProducts();
}

public interface IUserContext
{
    bool IsInRole(Role role);
    Currency Currency { get; }
}
```

Then, refactor `ProductService` to use Constructor Injection:

```csharp
public class ProductService
{
    private readonly IProductRepository repository;
    private readonly IUserContext userContext;

    public ProductService(
        IProductRepository repository,
        IUserContext userContext)
    {
        this.repository = repository ?? throw new ArgumentNullException(nameof(repository));
        this.userContext = userContext ?? throw new ArgumentNullException(nameof(userContext));
    }

    public IEnumerable<Product> GetFeaturedProducts()
    {
        return this.repository.GetFeaturedProducts()
            .Select(p => p.ApplyDiscountFor(this.userContext))
            .Select(p => p.ChangeCurrencyTo(this.userContext.Currency));
    }
}
```

Key technical aspects:
- **Constructor Injection**: Dependencies are passed via the constructor, ensuring they're required for the class to function. This is the authors' preferred pattern for mandatory dependencies, as it makes the class's requirements explicit and immutable.
- **Immutability**: Fields are `readonly`, preventing reassignment after construction (promotes thread-safety and clarity).
- **LINQ operations**: The method chains `Select` calls to apply discounts (based on user role) and convert currency. `ApplyDiscountFor` checks `userContext.IsInRole(Role.Vip)` for VIP discounts; `ChangeCurrencyTo` uses exchange rates (assumed to be handled in `Product` class).
- **Independence**: The domain is now free of infrastructure concerns (no SQL or HTTP references), making it reusable in non-web contexts (e.g., console apps).
- **Minor topic: Extension methods**: The authors note that `ApplyDiscountFor` and `ChangeCurrencyTo` could be extension methods on `Product`, but they keep them as instance methods for simplicity.

If currency conversion requires an external service (volatile), they'd abstract it as `ICurrencyConverter` and inject it too.

##### 3.1.3 Building a New Data Access Layer

The data access is volatile (might change from SQL to NoSQL), so it's abstracted behind `IProductRepository`. The concrete implementation:

```csharp
public class SqlProductRepository : IProductRepository
{
    private readonly string connectionString;

    public SqlProductRepository(string connectionString)
    {
        this.connectionString = connectionString ?? throw new ArgumentNullException(nameof(connectionString));
    }

    public IEnumerable<Product> GetFeaturedProducts()
    {
        using (var connection = new SqlConnection(this.connectionString))
        {
            connection.Open();
            using (var command = connection.CreateCommand())
            {
                command.CommandText = "SELECT * FROM [Product] WHERE [IsFeatured] = 1";
                using (var reader = command.ExecuteReader())
                {
                    var products = new List<Product>();
                    while (reader.Read())
                    {
                        products.Add(new Product
                        {
                            Id = (int)reader["Id"],
                            Name = (string)reader["Name"],
                            Description = (string)reader["Description"],
                            UnitPrice = new Money((Currency)Enum.Parse(typeof(Currency), (string)reader["Currency"]), (decimal)reader["Amount"]),
                            IsFeatured = true
                        });
                    }
                    return products;
                }
            }
        }
    }
}
```

Key technical aspects:
- **Constructor Injection for config**: The connection string is injected, avoiding hard-coded config (e.g., no `ConfigurationManager`).
- **ADO.NET usage**: Simple SQL query for featured products. The authors highlight manual mapping from `DbDataReader` to `Product` domain objects, emphasizing separation of concerns.
- **Disposable resources**: Proper `using` statements for connection and command to ensure resource cleanup.
- **Minor topic: Error handling**: In a real app, add try-catch for SQL exceptions, but omitted here for brevity. The authors warn against swallowing exceptions.

##### 3.1.4 Implementing an ASP.NET Core–Specific IUserContext Adapter

`IUserContext` abstracts user info (roles, currency preference). In ASP.NET Core, it's adapted from `HttpContext`:

```csharp
public class AspNetUserContextAdapter : IUserContext
{
    private static readonly Currency defaultCurrency = new Currency("USD");

    private readonly IHttpContextAccessor accessor;

    public AspNetUserContextAdapter(IHttpContextAccessor accessor)
    {
        this.accessor = accessor ?? throw new ArgumentNullException(nameof(accessor));
    }

    public bool IsInRole(Role role)
    {
        return this.accessor.HttpContext.User.IsInRole(role.ToString());
    }

    public Currency Currency
    get
    {
        string currencyCode = this.accessor.HttpContext.User?.FindFirst("PreferredCurrency")?.Value;
        if (Enum.TryParse<Currency>(currencyCode, out var currency))
        {
            return currency;
        }
        return defaultCurrency;
    }
}
```

Key technical aspects:
- **Adapter pattern**: Wraps `IHttpContextAccessor` (built-in in .NET Core) to abstract HTTP-specific details.
- **Fallback logic**: Defaults to USD if no preference is set.
- **Minor topic: Claims-based identity**: Uses `FindFirst` for currency, assuming user claims. The authors note this is extensible for other user data.
- **Injection**: `IHttpContextAccessor` is injected, avoiding static `HttpContext.Current` (Deprecated in .NET Core).

##### 3.1.5 Composing the Application in the Composition Root

The Composition Root is introduced as the place close to the application's entry point where the entire object graph is assembled. For pure DI (no container), they show manual wiring in a custom `CompositionRoot` class or in `Startup.cs`. In the book, they use a method in `Startup` for .NET Core compatibility, but to illustrate:

In Startup.cs (simplified for pure DI):

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // For pure DI, manual composition instead of AddTransient, etc.
        // But to show pure, assume a factory
    }

    // Composition Root method
    public void Compose()
    {
        var connectionString = this.Configuration["ConnectionStrings:CommerceConnectionString"];
        var repository = new SqlProductRepository(connectionString);
        var userContext = new AspNetUserContextAdapter(new HttpContextAccessor()); // Normally singleton
        var productService = new ProductService(repository, userContext);
        // For controllers, in .NET Core, we'd register with container, but for pure DI, use a custom IControllerActivator
    }
}
```

In the book, they explain that in ASP.NET Core, the built-in DI container is used later, but here they show the concept: assemble dependencies once, at the root.

Key technical aspects:
- **Composition Root definition**: One place for wiring, keeping other code clean of `new` keywords for dependencies.
- **Manual instantiation**: Shows the dependency graph explicitly.
- **Minor topic: Config injection**: Connection string from `IConfiguration`, injected into Startup.
- **For .NET Core**: They note that full integration uses `services.AddTransient<IProductRepository, SqlProductRepository>()`; but this is pure DI demo.

If a dependency like currency converter is added, it's injected here too.

#### 3.2 Analyzing the Loosely Coupled Implementation

##### 3.2.1 Understanding the Interaction Between Components

The authors diagram the flow: Browser -> HomeController (injected with ProductService) -> ProductService (injected with IProductRepository and IUserContext) -> SqlProductRepository and AspNetUserContextAdapter.

Interaction:
- High-level policy (domain logic) calls low-level mechanisms through abstractions.
- No cyclic dependencies.
- Each component has a single reason to change (SRP).

They introduce Method Injection as an alternative for dependencies that vary per call (e.g., if `GetFeaturedProducts` needed a parameter-specific dependency):

Example (not used in main app, but as illustration):

```csharp
public IEnumerable<Product> GetFeaturedProducts(Currency specificCurrency)
{
    // Method Injection for specificCurrency
    return this.repository.GetFeaturedProducts()
        .Select(p => p.ApplyDiscountFor(this.userContext))
        .Select(p => p.ChangeCurrencyTo(specificCurrency)); // Injected via method param
}
```

Key: Use for optional or per-operation dependencies; constructor for always-required.

##### 3.2.2 Analyzing the New Dependency Graph

The dependency graph is inverted: High-level modules depend on abstractions owned by them, implemented by low-level.

Diagram description (as in book): HomeController -> ProductService -> IProductRepository (implemented by SqlProductRepository), -> IUserContext (implemented by AspNetUserContextAdapter).

Benefits:
- **Testability**: Mock IProductRepository for unit tests (e.g., InMemoryRepository).
- **Extensibility**: Swap Sql to AzureTableRepository without changing domain/UI.
- **Parallel development**: Teams can work on domain without data layer ready.
- **Late binding**: Decide implementations at runtime/config.

Minor topic: Avoid over-abstraction; only abstract volatile parts.

#### Evaluation of the Loosely Coupled Application

The authors evaluate against Chapter 2's issues:
- **Composability improved**: Object graph assembled in one place; easy to reconfigure.
- **Maintainability**: Changes isolated (e.g., new DB doesn't affect domain).
- **Testability**: All seams for mocking.
- **Flexibility**: Supports OCP; add features via decorators later.

They note potential downsides: More code (interfaces), but benefits outweigh.

#### Summary

In this chapter, you've seen examples of Constructor Injection as the main way to achieve loose coupling. The authors also introduced Method Injection for per-method dependencies and the Composition Root as the centralized place for object composition. Property Injection is mentioned briefly (for optional dependencies, set via properties), but not used in the example—it's like `public IProductRepository Repository { get; set; }`, but they prefer constructors for required deps. The refactored app demonstrates how DI enables maintainable, extensible code. The authors' perspective: DI isn't just a technique; it's a mindset for managing complexity by separating concerns and inverting dependencies. If a concept like currency conversion isn't relatable here, consider a separate example: injecting an ILogger into a service for logging without hard-coding console output. Next chapters build on this with patterns and anti-patterns.
