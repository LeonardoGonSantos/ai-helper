# Clean Architecture & CQRS — Referência

## Regra de Dependência (The Dependency Rule)

```
                    ┌────────────────────────────────────────┐
                    │             Domain                      │
                    │  Entities, Value Objects, Domain Events │
                    │  Interfaces de repositório e serviços   │
                    │  ZERO dependências externas             │
                    └────────────────────┬───────────────────┘
                                         ↑
                    ┌────────────────────┴───────────────────┐
                    │           Application                   │
                    │  Commands, Queries, Handlers, DTOs      │
                    │  Validators, Use Cases                  │
                    │  Depende apenas de Domain               │
                    └────────────────────┬───────────────────┘
                                         ↑
               ┌─────────────────────────┴──────────────────────────┐
               │                                                      │
  ┌────────────┴────────────┐              ┌────────────────────────┐│
  │     Infrastructure      │              │     Presentation        ││
  │  EF Core, Repositories  │              │  Controllers, Endpoints ││
  │  HttpClients, Email      │              │  Middleware, SignalR    ││
  │  Cache, Migrations       │              │  Depende de Application ││
  └─────────────────────────┘              └────────────────────────┘│
                                                                      │
               └──────────────────────────────────────────────────────┘
```

**Setas = direção das dependências.** Código interno nunca depende de código externo.

---

## Violações mais comuns

### Violação 1 — EF Core annotations no Domain

```csharp
// ❌ RUIM — Domain depende de Infrastructure (EF Core)
using Microsoft.EntityFrameworkCore;          // ← Infra no Domain!

[Table("orders")]
public class Order
{
    [Key]
    [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
    public int Id { get; set; }

    [Required, MaxLength(100)]
    public string CustomerName { get; set; }

    [Column("order_date")]
    public DateTime OrderDate { get; set; }
}

// ✅ BOM — Domain limpo, configuração via Fluent API em Infrastructure
// Domain/Orders/Order.cs
public sealed class Order : IAggregateRoot
{
    public OrderId Id { get; private set; }
    public CustomerName CustomerName { get; private set; }
    public DateTime OrderDate { get; private set; }
    // Sem nenhum atributo de infra
}

// Infrastructure/Persistence/Configurations/OrderConfiguration.cs
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("orders");
        builder.HasKey(o => o.Id);
        builder.Property(o => o.Id)
            .HasConversion(id => id.Value, v => new OrderId(v));
        builder.OwnsOne(o => o.CustomerName, cn =>
            cn.Property(n => n.Value).HasColumnName("customer_name").HasMaxLength(100));
    }
}
```

### Violação 2 — Application usando Infrastructure diretamente

```csharp
// ❌ RUIM — Handler com acesso direto ao DbContext (Application → Infrastructure)
using Microsoft.EntityFrameworkCore; // Infra em Application!

public class GetOrderQueryHandler
{
    private readonly AppDbContext _context; // DbContext direto!

    public async Task<OrderDto?> Handle(GetOrderQuery query, CancellationToken ct)
        => await _context.Orders // Viola a Dependency Rule
            .Select(o => new OrderDto(o.Id, o.Total))
            .FirstOrDefaultAsync(o => o.Id == query.OrderId, ct);
}

// ✅ BOM — Handler usa interface definida em Application/Domain
public class GetOrderQueryHandler(IOrderReadRepository repo)
{
    public Task<OrderDto?> Handle(GetOrderQuery query, CancellationToken ct)
        => repo.GetProjectedAsync<OrderDto>(
            o => o.Id == query.OrderId,
            o => new OrderDto(o.Id, o.Total),
            ct);
}
// Interface IOrderReadRepository definida em Application
// Implementação SqlOrderReadRepository em Infrastructure
```

### Violação 3 — HttpContext em Application/Domain

```csharp
// ❌ RUIM — Application acoplada a ASP.NET
public class CreateOrderHandler
{
    private readonly IHttpContextAccessor _http; // ASP.NET em Application!

    public async Task Handle(CreateOrderCommand cmd, CancellationToken ct)
    {
        var userId = _http.HttpContext?.User.FindFirst("sub")?.Value; // Acoplamento!
        // ...
    }
}

// ✅ BOM — ICurrentUserService abstrai o contexto
// Application/Abstractions/ICurrentUserService.cs
public interface ICurrentUserService
{
    UserId? UserId { get; }
    bool IsAuthenticated { get; }
}

// Infrastructure/Services/CurrentUserService.cs (implementa com HttpContext)
public class CurrentUserService(IHttpContextAccessor accessor) : ICurrentUserService
{
    public UserId? UserId => accessor.HttpContext?.User.FindFirst("sub")?.Value is string id
        ? UserId.From(Guid.Parse(id)) : null;
    public bool IsAuthenticated => accessor.HttpContext?.User.Identity?.IsAuthenticated ?? false;
}

// Handler limpo:
public class CreateOrderHandler(ICurrentUserService currentUser)
{
    public async Task Handle(CreateOrderCommand cmd, CancellationToken ct)
    {
        if (currentUser.UserId is null) throw new UnauthorizedException();
        // ...
    }
}
```

---

## CQRS — separação Command/Query

```csharp
// Convenção clara de nomenclatura:
// Commands: CreateOrder, ConfirmOrder, CancelOrder (mudam estado)
// Queries: GetOrderById, GetOrdersByCustomer (apenas leitura)

// Command — muda estado, retorna Result (não dados)
public record CreateOrderCommand(
    CustomerId CustomerId,
    IReadOnlyList<OrderItemDto> Items,
    Address ShippingAddress) : ICommand<Result<OrderId>>;

public class CreateOrderHandler(
    IOrderRepository repo,
    IUnitOfWork unitOfWork) : ICommandHandler<CreateOrderCommand, Result<OrderId>>
{
    public async Task<Result<OrderId>> HandleAsync(
        CreateOrderCommand cmd, CancellationToken ct)
    {
        var orderResult = Order.Create(cmd.CustomerId);
        if (orderResult.IsFailure) return orderResult.Error;

        var order = orderResult.Value;
        foreach (var item in cmd.Items)
            order.AddItem(item.ProductId, item.Price, item.Quantity);

        repo.Add(order);
        await unitOfWork.SaveChangesAsync(ct);
        return order.Id;
    }
}

// Query — leitura otimizada, pode pular domain model
public record GetOrderByIdQuery(OrderId OrderId) : IQuery<Result<OrderDetailDto>>;

public class GetOrderByIdHandler(IOrderReadRepository repo)
    : IQueryHandler<GetOrderByIdQuery, Result<OrderDetailDto>>
{
    public async Task<Result<OrderDetailDto>> HandleAsync(
        GetOrderByIdQuery query, CancellationToken ct)
    {
        var dto = await repo.GetDetailAsync(query.OrderId, ct);
        return dto is null
            ? Result.Failure<OrderDetailDto>(DomainErrors.Order.NotFound)
            : Result.Success(dto);
    }
}
```

---

## Fat Controller — sinalizar e refatorar

```csharp
// ❌ RUIM — Controller fazendo tudo (100+ linhas)
[HttpPost("checkout")]
public async Task<IActionResult> Checkout([FromBody] CheckoutRequest req)
{
    // Validação manual
    if (req.Items == null || !req.Items.Any())
        return BadRequest("No items");
    if (req.ShippingAddress == null)
        return BadRequest("No address");

    // Lógica de negócio no controller!
    decimal total = 0;
    foreach (var item in req.Items)
    {
        var product = await _context.Products.FindAsync(item.ProductId);
        if (product == null) return NotFound($"Product {item.ProductId} not found");
        if (product.Stock < item.Quantity) return BadRequest("Insufficient stock");
        product.Stock -= item.Quantity;
        total += product.Price * item.Quantity;
    }

    var order = new Order
    {
        CustomerId = req.CustomerId,
        Total = total,
        Status = "pending"
    };
    _context.Orders.Add(order);
    await _context.SaveChangesAsync();

    // Side effects diretos
    await _emailService.SendConfirmationAsync(req.CustomerEmail, order.Id);
    await _inventoryService.ReserveAsync(req.Items);

    return Ok(new { order.Id, total });
}

// ✅ BOM — Controller thin, delega tudo
[HttpPost("checkout")]
public async Task<IActionResult> Checkout(
    CheckoutCommand command, CancellationToken ct)
{
    var result = await _dispatcher.SendAsync(command, ct);
    return result.Match(
        orderId => CreatedAtAction(nameof(GetOrder), new { id = orderId }, orderId),
        error => Problem(error.Description, statusCode: error.Code switch
        {
            "NotFound" => 404,
            "InsufficientStock" => 422,
            _ => 400
        })
    );
}
```

---

## Vertical Slice Architecture — organização por feature

```
// ❌ RUIM — organização por camada técnica (difícil encontrar feature completa)
src/
├── Controllers/
│   ├── OrdersController.cs
│   └── ProductsController.cs
├── Services/
│   ├── OrderService.cs
│   └── ProductService.cs
├── Repositories/
│   ├── OrderRepository.cs
│   └── ProductRepository.cs
└── Models/
    ├── Order.cs
    └── Product.cs

// ✅ BOM — organização por feature (todo contexto numa pasta)
src/
├── Features/
│   ├── Orders/
│   │   ├── CreateOrder/
│   │   │   ├── CreateOrderCommand.cs
│   │   │   ├── CreateOrderHandler.cs
│   │   │   ├── CreateOrderValidator.cs
│   │   │   ├── CreateOrderEndpoint.cs
│   │   │   └── CreateOrderTests.cs   ← testes junto!
│   │   └── GetOrderById/
│   │       ├── GetOrderByIdQuery.cs
│   │       ├── GetOrderByIdHandler.cs
│   │       └── GetOrderByIdEndpoint.cs
│   └── Products/
│       └── ...
└── Domain/   ← Core domain permanece separado
    ├── Orders/
    └── Products/
```
