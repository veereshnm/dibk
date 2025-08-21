## Chapter 5: Dependency Injection in ASP.NET Core

Chapter 5 explores how ASP.NET Core integrates Dependency Injection as a first-class citizen, leveraging its built-in DI container to manage dependencies in a web application. The authors emphasize that ASP.NET Core is designed with DI at its core, making it a natural fit for applying the principles and patterns discussed in earlier chapters (e.g., Constructor Injection, Composition Root). This chapter provides practical guidance on using the DI system in ASP.NET Core, covering service registration, lifetime management, advanced scenarios, and common pitfalls. It assumes familiarity with basic DI concepts from previous chapters and focuses on their application within the ASP.NET Core framework.

### 5.1 Introduction to DI in ASP.NET Core

**Key Points:**
- ASP.NET Core includes a built-in DI container, often referred to as the *Microsoft.Extensions.DependencyInjection* container, which is simple yet sufficient for most application needs.
- The DI container is integrated into the ASP.NET Core pipeline, enabling dependency injection for controllers, services, middleware, and other components.
- The chapter emphasizes that DI in ASP.NET Core follows the *Constructor Injection* pattern, aligning with the principles of explicit dependencies and loose coupling.
- The authors highlight that while the built-in container is adequate for most scenarios, it lacks some advanced features (e.g., auto-registration, interception) found in third-party containers like Autofac or SimpleInjector. However, the built-in container can be replaced if needed.

**Authors’ Perspective:**
The authors stress that ASP.NET Core’s DI system is designed to be lightweight and easy to use, encouraging developers to adopt DI without needing external libraries for most applications. They advocate for leveraging the framework’s conventions to simplify DI configuration while adhering to SOLID principles.

### 5.2 Setting Up the Composition Root

The Composition Root is the centralized location where all dependencies are composed. In ASP.NET Core, the Composition Root is typically configured in the `Startup` class (or, in later versions, the `Program.cs` file for top-level statements).

#### 5.2.1 Configuring Services in `Startup`

**Book Example:**
The book uses a sample e-commerce application called *Commerce*, which includes a product catalog and pricing logic. The Composition Root is set up in the `Startup` class, specifically in the `ConfigureServices` method.

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllersWithViews();
        services.AddScoped<IProductService, ProductService>();
        services.AddScoped<IProductRepository, SqlProductRepository>();
        services.AddScoped<IPricingService, PricingService>();
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        // Pipeline configuration (omitted for brevity)
    }
}
```

**Explanation:**
- `IServiceCollection` is the interface used to register services with the DI container.
- `AddScoped`, `AddTransient`, and `AddSingleton` are extension methods to register services with different lifetimes (discussed later).
- In the example, `IProductService`, `IProductRepository`, and `IPricingService` are interfaces implemented by concrete classes (`ProductService`, `SqlProductRepository`, `PricingService`). These are registered with the *Scoped* lifetime, meaning a new instance is created per HTTP request.
- `AddControllersWithViews` registers the MVC framework, which relies on the DI container to resolve controllers and their dependencies.

**Technical Details:**
- The `ConfigureServices` method is called during application startup, building the service provider (`IServiceProvider`) that resolves dependencies at runtime.
- The authors emphasize that the `Startup` class is the ideal place for the Composition Root because it centralizes dependency configuration, making it easy to understand and maintain.
- The book warns against scattering dependency registrations across the application, as this violates the Composition Root pattern and makes the system harder to reason about.

#### 5.2.2 Program.cs in .NET 6 and Later

For .NET 6 and later, the authors note that the `Startup` class is optional, and the Composition Root can be configured directly in `Program.cs` using top-level statements. For example:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllersWithViews();
builder.Services.AddScoped<IProductService, ProductService>();
builder.Services.AddScoped<IProductRepository, SqlProductRepository>();
builder.Services.AddScoped<IPricingService, PricingService>();

var app = builder.Build();
app.UseRouting();
app.MapControllerRoute("default", "{controller=Home}/{action=Index}/{id?}");
app.Run();
```

**Authors’ Perspective:**
The shift to `Program.cs` simplifies the setup but doesn’t change the underlying DI principles. The authors stress that whether using `Startup` or `Program.cs`, the Composition Root should remain centralized and explicit.

### 5.3 Service Lifetimes

ASP.NET Core supports three primary service lifetimes: **Transient**, **Scoped**, and **Singleton**. The authors dedicate significant attention to explaining these lifetimes, as choosing the correct lifetime is critical for application correctness and performance.

#### 5.3.1 Transient Lifetime

- **Definition:** A new instance is created each time the service is resolved.
- **Registration:** `services.AddTransient<TService, TImplementation>();`
- **Use Case:** Suitable for lightweight, stateless services where a new instance per resolution is acceptable.

**Book Example:**
```csharp
services.AddTransient<IPriceCalculator, DiscountedPriceCalculator>();
```

**Explanation:**
- In the *Commerce* application, `IPriceCalculator` is a stateless service that calculates prices based on input. Since it has no state, creating a new instance per resolution (e.g., per controller action) is safe and efficient.
- The authors note that Transient services are short-lived and garbage-collected quickly, but overuse can lead to performance issues due to frequent object creation.

#### 5.3.2 Scoped Lifetime

- **Definition:** A single instance is created per scope, typically an HTTP request in ASP.NET Core.
- **Registration:** `services.AddScoped<TService, TImplementation>();`
- **Use Case:** Ideal for services that maintain state for the duration of a request, such as database contexts or services that share data within a request.

**Book Example:**
```csharp
services.AddScoped<IProductRepository, SqlProductRepository>();
```

**Explanation:**
- `SqlProductRepository` interacts with a database and may cache data or maintain a connection for the duration of a request. Using *Scoped* ensures that all components within the same HTTP request share the same instance.
- The authors highlight that ASP.NET Core automatically creates a scope per HTTP request, making *Scoped* the default choice for most application services.

#### 5.3.3 Singleton Lifetime

- **Definition:** A single instance is created for the entire application lifetime and shared across all requests.
- **Registration:** `services.AddSingleton<TService, TImplementation>();`
- **Use Case:** Suitable for stateless services or services that maintain global state, such as configuration or caching services.

**Book Example:**
```csharp
services.AddSingleton<IConfiguration, Configuration>();
```

**Explanation:**
- `IConfiguration` holds application-wide settings, which are loaded once at startup and remain constant. Using *Singleton* ensures a single instance is shared across all requests.
- The authors warn that Singletons must be thread-safe, as they are accessed concurrently by multiple requests.

**Technical Considerations:**
- The book provides a detailed comparison of lifetimes, emphasizing that choosing the wrong lifetime can lead to bugs (e.g., sharing a Scoped service across requests or using a Transient service for stateful operations).
- The authors introduce the concept of *Captive Dependencies*, where a service with a longer lifetime (e.g., Singleton) depends on a service with a shorter lifetime (e.g., Scoped). This can cause issues, such as a Singleton holding onto a Scoped database context, leading to incorrect behavior or resource leaks.
- **Example of Captive Dependency Issue:**
  ```csharp
  services.AddSingleton<IProductService, ProductService>();
  services.AddScoped<IProductRepository, SqlProductRepository>();
  ```
  Here, `ProductService` (Singleton) depends on `SqlProductRepository` (Scoped). Since the Singleton is created once, it captures the Scoped repository from the first request’s scope, reusing it across all requests, which can lead to stale data or concurrency issues.

**Authors’ Perspective:**
The authors strongly recommend validating lifetime configurations to avoid Captive Dependencies. They suggest using tools like SimpleInjector’s diagnostic features (if using a third-party container) or manually reviewing registrations. For the built-in container, developers must be diligent, as it lacks built-in diagnostics.

### 5.4 Dependency Injection in Controllers

ASP.NET Core controllers are resolved by the DI container, and dependencies are injected via constructor parameters.

**Book Example:**
```csharp
public class ProductController : Controller
{
    private readonly IProductService _productService;

    public ProductController(IProductService productService)
    {
        _productService = productService ?? throw new ArgumentNullException(nameof(productService));
    }

    public IActionResult Index()
    {
        var products = _productService.GetFeaturedProducts();
        return View(products);
    }
}
```

**Explanation:**
- The `ProductController` depends on `IProductService`, which is injected via Constructor Injection.
- The DI container resolves `IProductService` and its dependencies (e.g., `IProductRepository`, `IPricingService`) when creating the controller.
- The authors emphasize null checks in constructors to ensure dependencies are provided, aligning with defensive programming practices.
- The `Index` action uses `_productService` to fetch featured products, demonstrating how DI enables loose coupling between the controller and business logic.

**Technical Details:**
- Controllers are typically registered with *Transient* lifetime by the `AddControllersWithViews` method, but their dependencies (e.g., `IProductService`) can have different lifetimes (e.g., Scoped).
- The authors note that ASP.NET Core’s DI system automatically resolves controller dependencies, making it seamless to apply DI in MVC applications.

### 5.5 Advanced Scenarios

#### 5.5.1 Injecting Dependencies into Middleware

Middleware components in ASP.NET Core can also use DI. The authors explain two approaches: constructor injection and per-request injection.

**Book Example (Constructor Injection in Middleware):**
```csharp
public class LoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger _logger;

    public LoggingMiddleware(RequestDelegate next, ILogger logger)
    {
        _next = next;
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }

    public async Task InvokeAsync(HttpContext context)
    {
        _logger.Log("Request received");
        await _next(context);
    }
}
```

**Registration:**
```csharp
services.AddSingleton<ILogger, ConsoleLogger>();
```

**Explanation:**
- The middleware depends on `ILogger`, injected via the constructor.
- The `RequestDelegate` parameter allows the middleware to call the next component in the pipeline.
- Since middleware instances are created once at startup, their dependencies (e.g., `ILogger`) should be Singleton or carefully managed to avoid Captive Dependencies.

**Per-Request Injection:**
For dependencies with shorter lifetimes (e.g., Scoped), the authors recommend using the `InvokeAsync` method to receive dependencies.

**Example:**
```csharp
public class LoggingMiddleware
{
    private readonly RequestDelegate _next;

    public LoggingMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context, IProductRepository repository)
    {
        // Use repository (Scoped) within the request
        await _next(context);
    }
}
```

**Explanation:**
- The `IProductRepository` is injected into `InvokeAsync`, ensuring it’s resolved per request.
- This approach avoids Captive Dependencies when the middleware needs Scoped services.

#### 5.5.2 Factory-Based Registration

The authors discuss scenarios where concrete instances need to be created dynamically, using factory methods or delegates.

**Book Example:**
```csharp
services.AddScoped<IProductService>(sp =>
{
    var repository = sp.GetService<IProductRepository>();
    var pricingService = sp.GetService<IPricingService>();
    return new ProductService(repository, pricingService);
});
```

**Explanation:**
- The lambda expression acts as a factory, allowing custom logic for creating `ProductService`.
- `sp` is the `IServiceProvider`, used to resolve dependencies (`IProductRepository`, `IPricingService`) within the factory.
- This is useful when the service creation requires additional logic or parameterization not handled by simple constructor injection.

#### 5.5.3 Replacing the Built-In Container

While the built-in DI container is sufficient for most applications, the authors note that advanced scenarios (e.g., auto-registration, interception, or diagnostics) may require a third-party container like Autofac or SimpleInjector.

**Example with Autofac:**
```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllersWithViews();
    }

    public void ConfigureContainer(ContainerBuilder builder)
    {
        builder.RegisterType<ProductService>().As<IProductService>().InstancePerLifetimeScope();
        builder.RegisterType<SqlProductRepository>().As<IProductRepository>().InstancePerLifetimeScope();
        builder.RegisterType<PricingService>().As<IPricingService>().InstancePerLifetimeScope();
    }
}
```

**Explanation:**
- The `ConfigureContainer` method is used to configure Autofac, which replaces the built-in container.
- `InstancePerLifetimeScope` in Autofac is equivalent to ASP.NET Core’s Scoped lifetime.
- The authors caution that replacing the container requires careful integration to ensure compatibility with ASP.NET Core’s conventions (e.g., controller resolution).

**Authors’ Perspective:**
The authors recommend sticking with the built-in container unless specific advanced features are needed, as it simplifies the application and aligns with ASP.NET Core’s conventions.

### 5.6 Common Pitfalls and Best Practices

The authors highlight several pitfalls to avoid and best practices to follow when using DI in ASP.NET Core.

#### 5.6.1 Pitfalls

1. **Service Locator Pattern:**
   - Using `IServiceProvider` directly in components (e.g., resolving services in a controller’s action method) is considered an anti-pattern because it hides dependencies and violates the Explicit Dependencies Principle.
   - **Example (Anti-Pattern):**
     ```csharp
     public IActionResult Index(IServiceProvider sp)
     {
         var service = sp.GetService<IProductService>();
         var products = service.GetFeaturedProducts();
         return View(products);
     }
     ```
   - **Correct Approach:** Use Constructor Injection instead.

2. **Captive Dependencies:**
   - As discussed earlier, mixing lifetimes incorrectly can lead to bugs. The authors recommend auditing registrations to ensure compatibility.

3. **Overusing Transient Lifetime:**
   - Overusing Transient can lead to performance issues due to excessive object creation. Scoped or Singleton lifetimes are often more appropriate for services with shared state or expensive initialization.

#### 5.6.2 Best Practices

1. **Use Constructor Injection:**
   - Always prefer Constructor Injection for controllers, services, and middleware to make dependencies explicit.

2. **Centralize the Composition Root:**
   - Keep all service registrations in `ConfigureServices` or `Program.cs` to maintain clarity and avoid scattered configuration.

3. **Validate Lifetimes:**
   - Regularly review service lifetimes to prevent Captive Dependencies. Consider using diagnostic tools if using a third-party container.

4. **Avoid Manual Service Resolution:**
   - Avoid calling `GetService` directly except in specific scenarios (e.g., factories or middleware `InvokeAsync`).

5. **Use Null Checks:**
   - Include null checks in constructors to ensure dependencies are provided, as shown in the `ProductController` example.

**Authors’ Perspective:**
The authors emphasize that following these best practices aligns with the principles of DI (e.g., SOLID, loose coupling) and ensures maintainable, testable code. They encourage developers to understand the DI container’s behavior deeply to avoid subtle bugs.

### 5.7 Additional Example (When Book Example Is Insufficient)

To clarify the concept of *Captive Dependencies* (not fully illustrated with a complex example in the book), consider the following scenario outside the *Commerce* application:

**Scenario:**
A `UserService` (Singleton) depends on a `DbContext` (Scoped). This creates a Captive Dependency because the Singleton captures the `DbContext` from the first request’s scope.

```csharp
public class UserService
{
    private readonly AppDbContext _dbContext;

    public UserService(AppDbContext dbContext)
    {
        _dbContext = dbContext;
    }

    public string GetUser(int id)
    {
        return _dbContext.Users.Find(id)?.Name;
    }
}

services.AddSingleton<UserService>();
services.AddDbContext<AppDbContext>(options => options.UseSqlServer(connectionString));
```

**Issue:**
- The `UserService` (Singleton) holds onto the first `AppDbContext` instance, which is disposed of at the end of the first request, leading to runtime errors (e.g., `ObjectDisposedException`) in subsequent requests.

**Solution:**
- Change `UserService` to Scoped or use a factory to resolve `AppDbContext` per request:
  ```csharp
  services.AddScoped<UserService>();
  ```
  Alternatively, use a factory:
  ```csharp
  services.AddSingleton<IUserService>(sp =>
  {
      return new UserService(() => sp.CreateScope().ServiceProvider.GetService<AppDbContext>());
  });

  public class UserService
  {
      private readonly Func<AppDbContext> _dbContextFactory;

      public UserService(Func<AppDbContext> dbContextFactory)
      {
          _dbContextFactory = dbContextFactory;
      }

      public string GetUser(int id)
      {
          using var dbContext = _dbContextFactory();
          return dbContext.Users.Find(id)?.Name;
      }
  }
  ```

**Explanation:**
- The factory approach ensures a new `AppDbContext` is created per request, avoiding the Captive Dependency issue.
- This example reinforces the authors’ warning about lifetime mismatches and demonstrates a practical solution.

### 5.8 Summary of Authors’ Perspective

The authors view ASP.NET Core’s DI system as a practical, lightweight solution that encourages good DI practices (e.g., Constructor Injection, centralized Composition Root) while being accessible to developers. They stress the importance of understanding service lifetimes to avoid common pitfalls like Captive Dependencies. The chapter balances practical guidance (e.g., code examples in the *Commerce* application) with theoretical alignment to DI principles, ensuring developers can apply these concepts effectively in real-world ASP.NET Core applications. The authors also acknowledge the limitations of the built-in container and provide guidance on when and how to integrate third-party containers.

### 5.9 Minor Topics and Nuances

- **Named Services:** The built-in container doesn’t support named services natively, unlike some third-party containers. The authors suggest using factories or wrapper classes for scenarios requiring multiple implementations of the same interface.
- **IDisposable Dependencies:** Services implementing `IDisposable` (e.g., `DbContext`) are automatically disposed of by the container at the end of their lifetime (e.g., end of request for Scoped services).
- **ServiceProvider Scope:** The authors clarify that `IServiceProvider` obtained from `BuildServiceProvider` creates a new scope, and developers should avoid creating unnecessary scopes to prevent resource leaks.
- **Testing Considerations:** The chapter briefly mentions that DI simplifies unit testing by allowing dependencies to be mocked, but this is explored further in later chapters.

---

## Conclusion

Chapter 5 of *Dependency Injection Principles, Practices, and Patterns* provides a thorough guide to using DI in ASP.NET Core, leveraging the built-in container to implement robust, maintainable applications. By focusing on the *Commerce* application, the authors demonstrate practical applications of Constructor Injection, service lifetimes, and the Composition Root. They address advanced scenarios (e.g., middleware, factories) and highlight pitfalls like Captive Dependencies, ensuring developers understand both the “how” and “why” of DI in ASP.NET Core. The additional example provided here clarifies complex concepts like Captive Dependencies, aligning with the authors’ emphasis on lifetime management.

If you have specific questions about any section, code example, or concept from Chapter 5, or if you’d like me to dive deeper into a particular topic (e.g., integrating a third-party container or testing with DI), please let me know!