# Design Patterns — Referência C# com exemplos

## Quando sugerir cada pattern (regra rápida)

| Smell detectado | Pattern sugerido |
|---|---|
| 3+ if/else ou switch com lógica distinta | **Strategy** |
| `new ConcreteClass()` em lógica de negócio | **Factory / Abstract Factory** |
| Construtor com 5+ parâmetros | **Builder** |
| Logging + caching + retry no mesmo service | **Decorator** |
| Services com 5+ responsabilidades | **Mediator / CQRS** |
| GetActiveX, GetAdminX, GetArchivedX no repo | **Specification** |
| Objeto que pode ter estado nulo | **Null Object** |
| Precisar notificar múltiplos componentes | **Observer / Domain Events** |

---

## Strategy Pattern

**Detectar:** `switch` ou `if/else if` com 3+ branches onde cada branch tem lógica diferente (não apenas mapeamento simples).

```csharp
// ❌ RUIM — viola OCP, cada novo tipo = modificar o método
public decimal CalculateDiscount(string customerType, decimal amount) => customerType switch
{
    "Regular"  => amount * 0.05m,
    "Premium"  => amount * 0.10m,
    "VIP"      => amount * 0.20m,
    "Employee" => amount * 0.30m,
    _ => 0m
};
// Novo tipo "Partner" → alterar este método = risco de regressão

// ✅ BOM — Strategy via DI
public interface IDiscountStrategy
{
    string CustomerType { get; }
    decimal Calculate(decimal amount);
}

public class PremiumDiscountStrategy : IDiscountStrategy
{
    public string CustomerType => "Premium";
    public decimal Calculate(decimal amount) => amount * 0.10m;
}

public class DiscountCalculator
{
    private readonly Dictionary<string, IDiscountStrategy> _strategies;

    public DiscountCalculator(IEnumerable<IDiscountStrategy> strategies)
        => _strategies = strategies.ToDictionary(s => s.CustomerType);

    public decimal Calculate(string customerType, decimal amount)
        => _strategies.TryGetValue(customerType, out var s)
            ? s.Calculate(amount)
            : 0m;
}

// Registro: services.AddScoped<IDiscountStrategy, PremiumDiscountStrategy>();
// Novo tipo = novo arquivo, zero modificações em código existente
```

---

## Factory / Abstract Factory Pattern

**Detectar:** `new ConcreteType()` em Handler/Service/Domain; criação condicional de objetos; tight coupling a implementações.

```csharp
// ❌ RUIM
public class NotificationService
{
    public void Send(string type, string message)
    {
        if (type == "Email") new EmailSender().Send(message);   // Acoplado!
        else if (type == "Sms") new SmsSender().Send(message);  // Testável? Não.
        else if (type == "Push") new PushSender().Send(message);
    }
}

// ✅ BOM — Factory injected
public interface INotificationSenderFactory
{
    INotificationSender Create(NotificationType type);
}

public class NotificationSenderFactory(IServiceProvider sp) : INotificationSenderFactory
{
    public INotificationSender Create(NotificationType type) => type switch
    {
        NotificationType.Email => sp.GetRequiredService<EmailSender>(),
        NotificationType.Sms   => sp.GetRequiredService<SmsSender>(),
        NotificationType.Push  => sp.GetRequiredService<PushSender>(),
        _ => throw new ArgumentOutOfRangeException(nameof(type))
    };
}

public class NotificationService(INotificationSenderFactory factory)
{
    public Task SendAsync(NotificationType type, string message, CancellationToken ct)
        => factory.Create(type).SendAsync(message, ct);
}
```

---

## Builder Pattern

**Detectar:** construtores com 5+ parâmetros, objetos complexos construídos com muitos setters, ausência de test data builders em projetos de teste.

```csharp
// ❌ RUIM — 7 parâmetros, ordem fácil de trocar
var order = new Order(
    customerId, productId, 3, 49.99m, "BRL",
    new Address("Rua X", "São Paulo", "01310-100"),
    OrderStatus.Pending);

// ✅ BOM — Builder fluente
public class OrderBuilder
{
    private Guid _customerId = Guid.NewGuid();
    private List<(Guid ProductId, int Qty, Money Price)> _items = new();
    private Address _address = Address.Default;
    private OrderStatus _status = OrderStatus.Pending;

    public OrderBuilder ForCustomer(Guid customerId) { _customerId = customerId; return this; }
    public OrderBuilder WithItem(Guid productId, int qty, Money price)
    { _items.Add((productId, qty, price)); return this; }
    public OrderBuilder ShipTo(Address address) { _address = address; return this; }
    public OrderBuilder WithStatus(OrderStatus status) { _status = status; return this; }

    public Order Build() => new(_customerId, _items, _address, _status);
}

// Em testes:
var order = new OrderBuilder()
    .ForCustomer(CustomerId)
    .WithItem(ProductId, 3, Money.BRL(49.99m))
    .Build();
```

---

## Decorator Pattern

**Detectar:** método com logging + caching + retry + validação todos juntos; cross-cutting concerns misturados com lógica de negócio.

```csharp
// ❌ RUIM — 4 responsabilidades em um método
public async Task<Product> GetProductAsync(int id)
{
    _logger.LogInformation($"Getting product {id}"); // Logging
    if (_cache.TryGetValue($"product:{id}", out Product? cached)) return cached!; // Cache
    try
    {
        var product = await _repository.GetByIdAsync(id); // Negócio
        _cache.Set($"product:{id}", product, TimeSpan.FromMinutes(5)); // Cache novamente
        return product;
    }
    catch (Exception ex) { _logger.LogError(ex, "Error"); throw; }
}

// ✅ BOM — cada decorator tem uma responsabilidade
public class ProductService(IProductRepository repo) : IProductService
{
    public Task<Product?> GetByIdAsync(int id, CancellationToken ct)
        => repo.GetByIdAsync(id, ct); // Puro: só negócio
}

public class CachedProductService(IProductService inner, IMemoryCache cache) : IProductService
{
    public async Task<Product?> GetByIdAsync(int id, CancellationToken ct)
    {
        var key = $"product:{id}";
        if (cache.TryGetValue(key, out Product? p)) return p;
        p = await inner.GetByIdAsync(id, ct);
        if (p is not null) cache.Set(key, p, TimeSpan.FromMinutes(5));
        return p;
    }
}

// Registro com Scrutor:
services.AddScoped<IProductService, ProductService>()
        .Decorate<IProductService, CachedProductService>()
        .Decorate<IProductService, LoggingProductService>();
```

---

## Specification Pattern

**Detectar:** explosion de métodos de query no repository — `GetActive`, `GetByCategory`, `GetActiveByCategory`, `GetPremium`...

```csharp
// ❌ RUIM — repository com método para cada combinação de filtro
public interface IProductRepository
{
    Task<List<Product>> GetActiveAsync();
    Task<List<Product>> GetActiveByCategoryAsync(string category);
    Task<List<Product>> GetActivePremiumAsync();
    Task<List<Product>> GetActivePremiumByCategoryAsync(string category);
    // Cresce indefinidamente...
}

// ✅ BOM — Specification composable
public abstract class Specification<T>
{
    public abstract Expression<Func<T, bool>> ToExpression();
    public bool IsSatisfiedBy(T entity) => ToExpression().Compile()(entity);

    public Specification<T> And(Specification<T> other) => new AndSpec<T>(this, other);
    public Specification<T> Or(Specification<T> other)  => new OrSpec<T>(this, other);
}

public class ActiveProductSpec : Specification<Product>
{
    public override Expression<Func<Product, bool>> ToExpression()
        => p => p.IsActive && !p.IsDeleted;
}

public class CategorySpec(string category) : Specification<Product>
{
    public override Expression<Func<Product, bool>> ToExpression()
        => p => p.Category == category;
}

// Uso:
var spec = new ActiveProductSpec().And(new CategorySpec("Electronics"));
var products = await repo.FindAsync(spec, ct);
```

---

## CQRS sem MediatR

**⚠️ ALERTA:** MediatR ≥ 13.0 é licença comercial (Lucky Penny Software, abril 2025).
Versões ≤ 12.x: MIT. Verificar no `.csproj` e `packages.lock.json`.

**Alternativas open source:**
- **Wolverine** (`WolverineFx.Wolverine`) — convention-based, transactional outbox, MIT
- **martinothamar/Mediator** — source generator, ~zero overhead, Native AOT, MIT
- **Implementação própria** (abaixo):

```csharp
// Interfaces mínimas
public interface ICommand<TResult> { }
public interface IQuery<TResult> { }

public interface ICommandHandler<TCommand, TResult>
    where TCommand : ICommand<TResult>
{
    Task<TResult> HandleAsync(TCommand command, CancellationToken ct = default);
}

// Dispatcher
public class Dispatcher(IServiceProvider sp)
{
    public Task<TResult> SendAsync<TResult>(
        ICommand<TResult> command, CancellationToken ct = default)
    {
        var handlerType = typeof(ICommandHandler<,>)
            .MakeGenericType(command.GetType(), typeof(TResult));
        dynamic handler = sp.GetRequiredService(handlerType);
        return handler.HandleAsync((dynamic)command, ct);
    }
}

// Wolverine (recomendado — zero interfaces necessárias):
public class CreateOrderHandler
{
    public async Task<OrderId> Handle(CreateOrder command, AppDbContext db)
    {
        var order = Order.Create(command.CustomerId, command.Items);
        db.Orders.Add(order);
        await db.SaveChangesAsync();
        return order.Id;
    }
}
```

---

## Null Object Pattern

**Detectar:** null checks repetidos (`if (logger != null) logger.Log(...)`) ou retorno de `null` onde comportamento vazio faria mais sentido.

```csharp
// ❌ RUIM
if (_auditLogger != null)
    await _auditLogger.LogAsync(action);

// ✅ BOM
public class NullAuditLogger : IAuditLogger
{
    public Task LogAsync(AuditAction action) => Task.CompletedTask; // No-op
}
// Register: services.AddSingleton<IAuditLogger, NullAuditLogger>() para ambientes sem auditoria
```
