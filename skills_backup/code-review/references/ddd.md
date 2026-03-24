# DDD — Domain-Driven Design Reference

## Value Objects — eliminar Primitive Obsession

### Quando sugerir Value Object

| Código visto | Value Object a sugerir |
|---|---|
| `string email` | `Email` |
| `string cpf` / `string cnpj` | `Cpf`, `Cnpj` |
| `decimal amount` sem contexto | `Money(decimal Amount, string Currency)` |
| `int orderId` e `int customerId` no mesmo escopo | `OrderId`, `CustomerId` (strong-typed IDs) |
| `string street, string city, string zipCode` juntos | `Address` |
| `string status` ou `int status` | `OrderStatus` enum ou Value Object |
| `string phone` | `PhoneNumber` |

### Implementação com records

```csharp
// ❌ RUIM — string sem validação, sem semântica
public class Customer
{
    public string Email { get; set; }       // Pode ser "nao-e-email"
    public string Cpf { get; set; }         // Pode ser "123"
    public decimal Balance { get; set; }    // USD? BRL? Negativo?
}

// ✅ BOM — Value Objects validados
public record Email
{
    public string Value { get; }
    private Email(string v) => Value = v;

    public static Result<Email> Create(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            return Result.Failure<Email>(DomainErrors.Email.Empty);
        if (!Regex.IsMatch(value, @"^[^@\s]+@[^@\s]+\.[^@\s]+$"))
            return Result.Failure<Email>(DomainErrors.Email.InvalidFormat);
        return Result.Success(new Email(value.ToLowerInvariant()));
    }

    public override string ToString() => Value;
}

public record Money(decimal Amount, string Currency)
{
    public static Money Zero(string currency) => new(0, currency);

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new DomainException($"Cannot add {Currency} and {other.Currency}");
        return this with { Amount = Amount + other.Amount };
    }

    public Money Multiply(decimal factor) => this with { Amount = Amount * factor };
}

// Strong-typed IDs (evita confusão entre IDs de diferentes aggregates)
public readonly record struct OrderId(Guid Value)
{
    public static OrderId New() => new(Guid.NewGuid());
    public static OrderId From(Guid value) => new(value);
    public override string ToString() => Value.ToString();
}
```

---

## Aggregate Root — protegendo invariantes

```csharp
// ❌ RUIM — entidade anêmica com tudo público
public class Order
{
    public Guid Id { get; set; }
    public OrderStatus Status { get; set; }      // Qualquer um muda!
    public List<OrderItem> Items { get; set; }   // Qualquer um adiciona/remove!
    public decimal Total { get; set; }           // Calculado fora?
}

// Uso externo que viola invariantes:
order.Items.Add(new OrderItem { ProductId = id, Quantity = -1 }); // Qty negativa!
order.Status = OrderStatus.Delivered;  // Pulando pagamento!
order.Total = 999999m;                 // Total arbitrário!

// ✅ BOM — Aggregate Root com invariantes
public sealed class Order : IAggregateRoot
{
    private readonly List<OrderItem> _items = new();
    private readonly List<IDomainEvent> _domainEvents = new();

    public OrderId Id { get; private set; }
    public CustomerId CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public Money Total => _items.Aggregate(
        Money.Zero("BRL"), (sum, i) => sum.Add(i.Subtotal));

    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    // Factory method — garante estado válido desde a criação
    public static Result<Order> Create(CustomerId customerId)
    {
        if (customerId == default)
            return Result.Failure<Order>(DomainErrors.Order.InvalidCustomer);

        var order = new Order
        {
            Id = OrderId.New(),
            CustomerId = customerId,
            Status = OrderStatus.Draft
        };
        order._domainEvents.Add(new OrderCreatedDomainEvent(order.Id));
        return Result.Success(order);
    }

    public Result AddItem(ProductId productId, Money unitPrice, int quantity)
    {
        if (Status != OrderStatus.Draft)
            return Result.Failure(DomainErrors.Order.CannotModifyConfirmed);
        if (quantity <= 0)
            return Result.Failure(DomainErrors.Order.InvalidQuantity);
        if (unitPrice.Amount <= 0)
            return Result.Failure(DomainErrors.Order.InvalidPrice);

        var existingItem = _items.FirstOrDefault(i => i.ProductId == productId);
        if (existingItem is not null)
            existingItem.IncreaseQuantity(quantity);
        else
            _items.Add(OrderItem.Create(productId, unitPrice, quantity));

        return Result.Success();
    }

    public Result Confirm()
    {
        if (Status != OrderStatus.Draft)
            return Result.Failure(DomainErrors.Order.AlreadyConfirmed);
        if (!_items.Any())
            return Result.Failure(DomainErrors.Order.NoItems);

        Status = OrderStatus.Confirmed;
        _domainEvents.Add(new OrderConfirmedDomainEvent(Id, CustomerId, Total));
        return Result.Success();
    }

    public void ClearDomainEvents() => _domainEvents.Clear();
}
```

---

## Anemic Domain Model — detectar e corrigir

### Sintomas no código

```csharp
// ❌ RUIM — 5 sintomas de Anemic Domain Model

// Sintoma 1: Apenas getters/setters
public class BankAccount
{
    public decimal Balance { get; set; }  // set público = sem controle
    public bool IsActive { get; set; }
    public List<Transaction> Transactions { get; set; } = new();
}

// Sintoma 2: Regras fora do domínio (em Service)
public class BankAccountService
{
    public void Deposit(BankAccount account, decimal amount)
    {
        if (!account.IsActive) throw new InvalidOperationException("Inactive account");
        if (amount <= 0) throw new ArgumentException("Amount must be positive");
        account.Balance += amount;  // Mutação direta de fora!
        account.Transactions.Add(new Transaction(amount, TransactionType.Credit));
    }

    public void Withdraw(BankAccount account, decimal amount)
    {
        if (!account.IsActive) throw new InvalidOperationException("Inactive account");
        if (amount > account.Balance) throw new InvalidOperationException("Insufficient funds");
        account.Balance -= amount;
        account.Transactions.Add(new Transaction(-amount, TransactionType.Debit));
    }
}

// ✅ BOM — Rich Domain Model
public sealed class BankAccount : IAggregateRoot
{
    private readonly List<Transaction> _transactions = new();

    public AccountId Id { get; private set; }
    public Money Balance { get; private set; }
    public bool IsActive { get; private set; }
    public IReadOnlyCollection<Transaction> Transactions => _transactions.AsReadOnly();

    public Result Deposit(Money amount)
    {
        if (!IsActive) return Result.Failure(DomainErrors.Account.Inactive);
        if (amount.Amount <= 0) return Result.Failure(DomainErrors.Account.InvalidAmount);

        Balance = Balance.Add(amount);
        _transactions.Add(Transaction.Credit(Id, amount));
        return Result.Success();
    }

    public Result Withdraw(Money amount)
    {
        if (!IsActive) return Result.Failure(DomainErrors.Account.Inactive);
        if (amount.Amount > Balance.Amount)
            return Result.Failure(DomainErrors.Account.InsufficientFunds);

        Balance = Balance with { Amount = Balance.Amount - amount.Amount };
        _transactions.Add(Transaction.Debit(Id, amount));
        return Result.Success();
    }
}
```

---

## Domain Events

**Detectar ausência quando:** após uma operação de domínio importante, outros serviços deveriam ser notificados mas são chamados diretamente no handler (acoplamento direto).

```csharp
// ❌ RUIM — Handler acoplado a múltiplos serviços pós-operação
public class ConfirmOrderHandler
{
    public async Task Handle(ConfirmOrderCommand cmd, CancellationToken ct)
    {
        var order = await _repo.GetByIdAsync(cmd.OrderId, ct);
        order.Status = OrderStatus.Confirmed; // Anêmico
        await _repo.SaveAsync(order, ct);

        // Acoplamentos diretos — violam SRP e OCP
        await _emailService.SendConfirmationAsync(order, ct);
        await _inventoryService.ReserveAsync(order.Items, ct);
        await _analyticsService.TrackOrderAsync(order, ct);
    }
}

// ✅ BOM — Domain Events desacoplam side effects
public class ConfirmOrderHandler
{
    public async Task Handle(ConfirmOrderCommand cmd, CancellationToken ct)
    {
        var order = await _repo.GetByIdAsync(cmd.OrderId, ct);
        order.Confirm(); // Levanta OrderConfirmedDomainEvent internamente
        await _unitOfWork.SaveChangesAsync(ct); // Dispatcher publica eventos
    }
}

// Handler de evento — independente, testável, substituível
public class SendConfirmationEmailOnOrderConfirmed(IEmailService email)
    : IDomainEventHandler<OrderConfirmedDomainEvent>
{
    public Task HandleAsync(OrderConfirmedDomainEvent @event, CancellationToken ct)
        => email.SendConfirmationAsync(@event.OrderId, @event.CustomerEmail, ct);
}
```

---

## Result Pattern — para falhas esperadas

```csharp
// ❌ RUIM — exception para fluxo esperado
public User GetById(int id)
{
    var user = _repo.Find(id);
    if (user is null) throw new UserNotFoundException(id); // Exceção para not found!
    return user;
}

// ✅ BOM — Result<T> para falhas de negócio esperadas
public record Error(string Code, string Description);

public class Result<T>
{
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public T? Value { get; }
    public Error? Error { get; }

    private Result(T value) { IsSuccess = true; Value = value; }
    private Result(Error error) { IsSuccess = false; Error = error; }

    public static Result<T> Success(T value) => new(value);
    public static Result<T> Failure(Error error) => new(error);

    public TResult Match<TResult>(Func<T, TResult> onSuccess, Func<Error, TResult> onFailure)
        => IsSuccess ? onSuccess(Value!) : onFailure(Error!);
}

// Uso:
public async Task<IResult> GetUser(int id, IUserRepository repo, CancellationToken ct)
{
    var result = await repo.GetByIdAsync(id, ct);
    return result.Match(
        user => Results.Ok(user),
        error => Results.NotFound(new { error.Code, error.Description })
    );
}
```
