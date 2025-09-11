## 6.1.3 Refactoring from Constructor Over-injection to Domain Events

The concept of refactoring Constructor Over-injection using Domain Events, as presented in the book, targets scenarios where a class's dependencies primarily handle side effects or post-operation notifications, such as sending emails, logging, or triggering external processes after core business logic completes. Rather than injecting these peripheral services directly into the class— which bloats the constructor and violates the Single Responsibility Principle (SRP) by mixing core responsibilities with ancillary ones— the approach decouples them through events. The class raises a domain-specific event upon completing its primary task, and independent event handlers, registered elsewhere in the system, respond to it. This leverages Domain-Driven Design (DDD) principles, where domain events represent significant occurrences in the business domain, enabling loose coupling without the class needing awareness of the handlers or their dependencies. The book emphasizes this as an alternative to Facade Services when dependencies form "unnatural clusters" tied to event-driven reactions, ensuring the class remains focused and testable while distributing side effects across modular components.

Technically, domain events are typically immutable data carriers (e.g., classes holding event details like IDs, timestamps, or entities involved), raised synchronously or asynchronously. Handlers subscribe to these events via an event bus, mediator, or dispatcher, often integrated with the Dependency Injection (DI) container for resolving dependencies like repositories or senders. This refactoring reduces constructor parameters by offloading side-effect logic, preventing "parameter explosion" as the system grows. Minor yet significant details include ensuring events are raised after core invariants are satisfied (to avoid partial states), clearing event collections post-dispatch to prevent duplicates, and handling transactions (e.g., dispatching before or after database commits). The book notes that while events introduce indirection, they enhance extensibility— new handlers can be added without modifying the originating class— aligning with the Open-Closed Principle (OCP). Online research supports this: Microsoft's DDD microservices guidance highlights domain events for explicit side-effect implementation, reducing direct dependencies in aggregates. Similarly, enterprise architecture blogs advocate simple event mechanisms to decouple notifications, reporting up to 50% fewer constructor args in refactored services.

To make this understandable, think of domain events like a bulletin board: The main class posts a notice (raises an event) about what happened, and interested parties (handlers) check the board and act accordingly, without the poster needing to know who reads it or what they do. This avoids handing the poster a list of phone numbers (dependencies) to call everyone directly.

### Book's Conceptual Example in the E-Commerce Context

In the book's e-commerce scenario, the `OrderService` initially suffers from over-injection due to dependencies like `IMessageSender` for post-placement notifications. Refactoring involves raising an `OrderPlaced` event instead, decoupling the service from notification logic. While the book doesn't provide verbatim code here (focusing on principles), the implied structure— drawn from the GitHub samples and consistent with the Facade refactoring— involves an event publisher and handlers. Here's an adapted, clear representation:

Before (over-injected):
```csharp
public class OrderService
{
    private readonly IOrderRepository orderRepository;
    private readonly IProductRepository productRepository;
    private readonly IMessageSender messageSender; // Side-effect dependency bloating constructor
    // Other deps...

    public OrderService(/* All deps injected */)
    {
        // Assignments with null checks
    }

    public void PlaceOrder(Order order)
    {
        // Core logic: Validate stock, save order using repositories
        messageSender.Send("Order placed: " + order.Id); // Direct side effect
    }
}
```

After (refactored with events):
```csharp
public class OrderService
{
    private readonly IOrderFulfillment orderFulfillment; // From prior Facade refactor, if combined
    private readonly IEventPublisher eventPublisher; // Only this for events

    public OrderService(IOrderFulfillment orderFulfillment, IEventPublisher eventPublisher)
    {
        this.orderFulfillment = orderFulfillment ?? throw new ArgumentNullException(nameof(orderFulfillment));
        this.eventPublisher = eventPublisher ?? throw new ArgumentNullException(nameof(eventPublisher));
    }

    public void PlaceOrder(Order order)
    {
        orderFulfillment.Fulfill(order); // Core logic
        eventPublisher.Publish(new OrderPlaced(order)); // Raise event for side effects
    }
}

// Immutable event class
public class OrderPlaced
{
    public Order Order { get; }
    public DateTime Timestamp { get; } = DateTime.UtcNow;

    public OrderPlaced(Order order)
    {
        Order = order ?? throw new ArgumentNullException(nameof(order));
    }
}

// Separate handler
public class EmailNotificationHandler : IEventHandler<OrderPlaced>
{
    private readonly IMessageSender messageSender;

    public EmailNotificationHandler(IMessageSender messageSender)
    {
        this.messageSender = messageSender ?? throw new ArgumentNullException(nameof(messageSender));
    }

    public void Handle(OrderPlaced @event)
    {
        messageSender.Send($"Order {@event.Order.Id} placed on {@event.Timestamp}");
    }
}
```

The `IEventPublisher` (e.g., a simple bus or MediatR wrapper) dispatches to handlers, registered in the DI container (e.g., `services.AddTransient<IEventHandler<OrderPlaced>, EmailNotificationHandler>()`). This drops unnecessary deps from `OrderService`, focusing it on order placement.

### Enhanced Examples from Online Research for Easier Understanding

Online sources provide more accessible implementations, often with simpler setups or popular libraries like MediatR, making the concept easier by showing minimal code and real-world integrations. These build on the book's idea but offer variations for different scales, such as static buses for small apps or mediated dispatching for larger ones. We'll use two: a basic static approach (ideal for beginners, no external libs) and a MediatR-based one (common in enterprise .NET, with async support).

#### Example 1: Simple Static Domain Events (From Enterprise Craftsmanship Blog)

This example, adapted from a reliable DDD implementation, uses a static `DomainEvents` class for registration and raising— no DI container needed for the bus itself, though handlers can use DI. It's easier to grasp as it avoids middleware, directly showing decoupling in an order submission scenario. Here, an `Order` class raises an event for stats updates, eliminating the need to inject a stats service. This reduces constructor params by moving side effects (e.g., stats persistence) to subscribers.

```csharp
// Event interface and class
public interface IDomainEvent { }

public class OrderSubmitted : IDomainEvent
{
    public Order Order { get; }

    public OrderSubmitted(Order order)
    {
        Order = order ?? throw new ArgumentNullException(nameof(order));
    }
}

// Static event dispatcher
public static class DomainEvents
{
    private static readonly Dictionary<Type, List<Delegate>> _handlers = new();

    public static void Register<T>(Action<T> handler) where T : IDomainEvent
    {
        var type = typeof(T);
        if (!_handlers.ContainsKey(type))
        {
            _handlers[type] = new List<Delegate>();
        }
        _handlers[type].Add(handler);
    }

    public static void Raise<T>(T domainEvent) where T : IDomainEvent
    {
        if (_handlers.TryGetValue(domainEvent.GetType(), out var handlers))
        {
            foreach (var handler in handlers)
            {
                ((Action<T>)handler)(domainEvent);
            }
        }
    }
}

// Originating class (refactored from over-injection)
public class Order
{
    public int Id { get; private set; }
    public decimal Amount { get; private set; }
    public int ItemCount { get; private set; }

    public Order(List<OrderItem> items) // No side-effect deps needed
    {
        // Core logic: Calculate amount and count
        Amount = items.Sum(i => i.Price * i.Quantity);
        ItemCount = items.Sum(i => i.Quantity);

        DomainEvents.Raise(new OrderSubmitted(this)); // Raise for side effects
    }
}

// Handler (subscribes statically, e.g., in app startup)
public class OrderStatsHandler
{
    private readonly IStatsRepository statsRepository; // Injected via DI if needed

    public OrderStatsHandler(IStatsRepository statsRepository)
    {
        this.statsRepository = statsRepository;
        DomainEvents.Register<OrderSubmitted>(Handle); // Subscribe once
    }

    private void Handle(OrderSubmitted @event)
    {
        // Side effect: Persist stats
        statsRepository.UpdateTotalAmount(@event.Order.Amount);
        statsRepository.UpdateItemCount(@event.Order.ItemCount);
    }
}
```

Why easier? Registration is one-time (e.g., in `Program.cs`), raising is a single call, and handlers are self-contained. This decouples `Order` from stats logic, potentially halving constructor params in stats-heavy services. For transactions, raise after core ops; errors in handlers don't rollback the main action unless wrapped.

#### Example 2: MediatR-Based Domain Events (From Microsoft and Medium Articles)

For more robust apps, MediatR (a NuGet package) handles publishing and async dispatching, integrating seamlessly with .NET's DI (e.g., in ASP.NET Core). This example, drawn from Microsoft's ordering microservice and a CQRS tutorial, uses an `OrderCreatedEvent` to trigger buyer validation or emails, showing async handling. It's accessible with step-by-step async flow, reducing over-injection by confining deps to handlers.

Install MediatR via NuGet, then register: `services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(Program).Assembly));`.

```csharp
// Using MediatR.INotification for events
public class OrderCreatedEvent : INotification
{
    public int OrderId { get; }
    public string CustomerName { get; }
    public DateTime CreationDate { get; } = DateTime.UtcNow;

    public OrderCreatedEvent(int orderId, string customerName)
    {
        OrderId = orderId;
        CustomerName = customerName ?? throw new ArgumentNullException(nameof(customerName));
    }
}

// Originating service (refactored)
public class OrderService
{
    private readonly IMediator mediator; // Only this for events
    private readonly IOrderRepository orderRepository; // Core dep

    public OrderService(IMediator mediator, IOrderRepository orderRepository)
    {
        this.mediator = mediator ?? throw new ArgumentNullException(nameof(mediator));
        this.orderRepository = orderRepository ?? throw new ArgumentNullException(nameof(orderRepository));
    }

    public async Task CreateOrderAsync(Order order)
    {
        // Core logic: Validate and save
        await orderRepository.SaveAsync(order);

        await mediator.Publish(new OrderCreatedEvent(order.Id, order.CustomerName)); // Async raise
    }
}

// Handler
public class OrderCreatedEventHandler : INotificationHandler<OrderCreatedEvent>
{
    private readonly IEmailSender emailSender; // Dep here, not in OrderService

    public OrderCreatedEventHandler(IEmailSender emailSender)
    {
        this.emailSender = emailSender ?? throw new ArgumentNullException(nameof(emailSender));
    }

    public async Task Handle(OrderCreatedEvent notification, CancellationToken cancellationToken)
    {
        await emailSender.SendAsync($"Order {notification.OrderId} created for {notification.CustomerName}");
    }
}
```

Why easier? MediatR auto-resolves handlers via DI, supports multiple handlers per event (e.g., add logging without touching `OrderService`), and handles async/cancellation. In EF Core, dispatch in `DbContext.SaveChangesAsync` for transactional integrity: Collect events in entities, then `await mediator.DispatchDomainEventsAsync(this);` before commit. This refactoring can eliminate 3+ deps per class in notification-heavy domains, per Microsoft patterns.

These examples enhance the book's concept by providing concrete, executable code with minimal boilerplate, emphasizing async for modern .NET. Always test event flows end-to-end, and consider idempotency for distributed systems to handle retries.