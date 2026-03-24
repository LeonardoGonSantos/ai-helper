# Resiliência, Retry, CancellationToken e Segurança

## Retry Policies — a regra mais violada

### Regra de ouro: NUNCA retry em 4xx

Erros 4xx são erros do **cliente** — o request é malformado, não autorizado, ou o recurso não existe. Retry jamais vai consertar um 400 Bad Request. Tentar novamente apenas desperdiça recursos e pode causar thundering herd.

```
Retry DEVE ser aplicado:              Retry NUNCA deve ser aplicado:
✅ 408 Request Timeout               ❌ 400 Bad Request
✅ 429 Too Many Requests (com delay) ❌ 401 Unauthorized
✅ 500 Internal Server Error         ❌ 403 Forbidden
✅ 502 Bad Gateway                   ❌ 404 Not Found
✅ 503 Service Unavailable           ❌ 405 Method Not Allowed
✅ 504 Gateway Timeout               ❌ 422 Unprocessable Entity
✅ Timeout de rede (IOException)     ❌ 409 Conflict (depende do domínio)
```

```csharp
// ❌ RUIM — retry em TODOS os status codes, incluindo 4xx!
var policy = Policy<HttpResponseMessage>
    .HandleResult(r => !r.IsSuccessStatusCode) // Inclui 400, 401, 403, 404!
    .WaitAndRetryAsync(3, _ => TimeSpan.FromSeconds(1));

// ✅ BOM — Polly v7 apenas para 5xx e timeouts
var policy = Policy<HttpResponseMessage>
    .Handle<HttpRequestException>()
    .OrResult(r => (int)r.StatusCode >= 500 || r.StatusCode == HttpStatusCode.RequestTimeout)
    .WaitAndRetryAsync(
        retryCount: 3,
        sleepDurationProvider: attempt =>
            TimeSpan.FromSeconds(Math.Pow(2, attempt)) // Exponential backoff
            + TimeSpan.FromMilliseconds(Random.Shared.Next(0, 1000))); // Jitter!

// ✅ MELHOR — .NET 8+ AddStandardResilienceHandler (configurado corretamente)
builder.Services.AddHttpClient<IExternalApiClient, ExternalApiClient>()
    .AddStandardResilienceHandler(options =>
    {
        options.Retry.ShouldHandle = args =>
            ValueTask.FromResult(
                args.Outcome.Exception is not null ||
                (int)(args.Outcome.Result?.StatusCode ?? 0) >= 500);
        options.Retry.MaxRetryAttempts = 3;
        options.Retry.Delay = TimeSpan.FromMilliseconds(500);
        options.Retry.BackoffType = DelayBackoffType.Exponential;
        options.Retry.UseJitter = true; // SEMPRE usar jitter
    });
```

---

## Circuit Breaker — evitar retry storms

Sem circuit breaker, um serviço degradado recebe uma enxurrada de retries → piora ainda mais.

```csharp
// ✅ BOM — Pipeline completo com retry + circuit breaker + timeout
builder.Services.AddResiliencePipeline("payment-api", builder =>
{
    // 1. Timeout por tentativa
    builder.AddTimeout(new TimeoutStrategyOptions
    {
        Timeout = TimeSpan.FromSeconds(5)
    });

    // 2. Retry com jitter (antes do circuit breaker)
    builder.AddRetry(new RetryStrategyOptions
    {
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromMilliseconds(500),
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true,
        ShouldHandle = new PredicateBuilder()
            .Handle<TimeoutRejectedException>()
            .Handle<HttpRequestException>()
            .HandleResult<HttpResponseMessage>(r => (int)r.StatusCode >= 500)
    });

    // 3. Circuit breaker (depois do retry)
    builder.AddCircuitBreaker(new CircuitBreakerStrategyOptions
    {
        FailureRatio = 0.5,               // Abre se 50% falhar
        SamplingDuration = TimeSpan.FromSeconds(30),
        MinimumThroughput = 10,           // Mínimo de requests para avaliar
        BreakDuration = TimeSpan.FromSeconds(15) // Fica aberto 15s
    });

    // 4. Timeout total (engloba tudo)
    builder.AddTimeout(TimeSpan.FromSeconds(30));
});
```

---

## CancellationToken — propagação obrigatória

### Checklist de propagação

```csharp
// ❌ RUIM — CancellationToken recebido mas ignorado
[HttpGet("{id}")]
public async Task<IActionResult> GetOrder(
    int id, CancellationToken cancellationToken) // Recebido do ASP.NET
{
    var order = await _context.Orders
        .FindAsync(id); // ct ignorado!
    var details = await _httpClient
        .GetAsync($"/api/details/{id}"); // ct ignorado!
    return Ok(order);
}

// ✅ BOM — CancellationToken propagado em toda a cadeia
[HttpGet("{id}")]
public async Task<IActionResult> GetOrder(int id, CancellationToken ct)
{
    var order = await _context.Orders.FindAsync([id], ct);
    if (order is null) return NotFound();

    var details = await _httpClient.GetAsync($"/api/details/{id}", ct);
    var dto = await details.Content.ReadFromJsonAsync<OrderDetails>(ct);

    return Ok(new OrderResponse(order, dto));
}

// ✅ BOM — propagar em services também
public class OrderService(AppDbContext db, IDetailsClient client)
{
    public async Task<OrderDto> GetAsync(int id, CancellationToken ct)
    {
        var order = await db.Orders
            .AsNoTracking()
            .Where(o => o.Id == id)
            .Select(o => new OrderDto(o.Id, o.Status))
            .FirstOrDefaultAsync(ct); // ct aqui

        return order ?? throw new NotFoundException(id);
    }
}
```

### CancellationToken em Background Services

```csharp
// ✅ BOM — usar stoppingToken do BackgroundService
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    while (!stoppingToken.IsCancellationRequested)
    {
        try
        {
            await ProcessBatchAsync(stoppingToken); // Propagado!
            await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
        }
        catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
        {
            break; // Shutdown gracioso
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Batch processing failed");
            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken); // Backoff em erro
        }
    }
}
```

---

## Exception Handling

### Throw vs Throw ex

```csharp
// ❌ RUIM — throw ex perde o stack trace original!
try { await ProcessAsync(); }
catch (Exception ex) { throw ex; } // Stack trace começa aqui!

// ✅ BOM — throw preserva stack trace original
try { await ProcessAsync(); }
catch (Exception ex)
{
    _logger.LogError(ex, "Processing failed");
    throw; // Stack trace completo preservado
}

// ✅ BOM — ExceptionDispatchInfo para re-throw em outro contexto
catch (Exception ex)
{
    var edi = ExceptionDispatchInfo.Capture(ex);
    // ... fazer algo ...
    edi.Throw(); // Preserva stack trace mesmo em outro contexto
}
```

### Hierarquia de exceções de domínio

```csharp
// ❌ RUIM — exceção genérica, impossível distinguir e tratar
throw new Exception("User not found");
throw new Exception("Insufficient funds");
throw new Exception("Order already confirmed");

// ✅ BOM — hierarquia clara e tratável
public abstract class DomainException(string code, string message)
    : Exception(message)
{
    public string Code { get; } = code;
}

public class NotFoundException(string entity, object id)
    : DomainException("NotFound", $"{entity} with id '{id}' was not found") { }

public class BusinessRuleException(string code, string message)
    : DomainException(code, message) { }

public class InsufficientFundsException(decimal requested, decimal available)
    : BusinessRuleException(
        "InsufficientFunds",
        $"Requested {requested} but only {available} available") { }

// Global exception handler (ASP.NET Core 8+)
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        var ex = context.Features.Get<IExceptionHandlerFeature>()?.Error;
        var (status, code) = ex switch
        {
            NotFoundException e    => (404, e.Code),
            BusinessRuleException e => (422, e.Code),
            _ => (500, "InternalError")
        };
        context.Response.StatusCode = status;
        await context.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Title = code,
            Detail = ex?.Message,
            Status = status
        });
    });
});
```

---

## Dependency Injection — Lifetimes e Captive Dependencies

### Tabela de lifetimes

```
Lifetime    | Criado quando       | Destruído quando    | Thread-safe?
Transient   | Cada injeção        | Fim do scope/GC     | Sim (nova instância)
Scoped      | Primeiro uso/scope  | Fim do HTTP request | Não (dentro do request)
Singleton   | Primeiro uso        | Fim da aplicação    | Deve ser!
```

### Captive Dependency — BUG SILENCIOSO

```csharp
// ❌ RUIM — Singleton captura Scoped = DbContext nunca é renovado!
services.AddSingleton<IReportGenerator, ReportGenerator>();
services.AddScoped<AppDbContext>(); // Scoped

public class ReportGenerator(AppDbContext context) : IReportGenerator
{
    // context foi criado uma vez e NUNCA muda
    // Change tracker acumula entidades infinitamente
    // Dados ficam stale
    // Thread-unsafe!
    public Report Generate() => new(context.Orders.ToList());
}

// ✅ BOM — IServiceScopeFactory para criar scope quando necessário
public class ReportGenerator(IServiceScopeFactory scopeFactory) : IReportGenerator
{
    public async Task<Report> GenerateAsync(CancellationToken ct)
    {
        await using var scope = scopeFactory.CreateAsyncScope();
        var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        var data = await context.Orders.AsNoTracking().ToListAsync(ct);
        return new Report(data);
    }
}

// Detectar em development:
builder.Host.UseDefaultServiceProvider(o =>
{
    o.ValidateScopes = true;   // Detecta captive dependencies
    o.ValidateOnBuild = true;  // Falha no startup se grafo inválido
});
```

---

## Memory Leaks comuns

### Event handlers não desregistrados

```csharp
// ❌ RUIM — MyComponent nunca será coletado enquanto EventSource existir
public class MyComponent
{
    public MyComponent(IEventSource source)
    {
        source.DataChanged += OnDataChanged; // Nunca desregistrado!
    }
    private void OnDataChanged(object? sender, EventArgs e) { }
}

// ✅ BOM — implementar IDisposable
public class MyComponent : IDisposable
{
    private readonly IEventSource _source;

    public MyComponent(IEventSource source)
    {
        _source = source;
        _source.DataChanged += OnDataChanged;
    }

    public void Dispose() => _source.DataChanged -= OnDataChanged;

    private void OnDataChanged(object? sender, EventArgs e) { }
}
```

### Streams não descartados

```csharp
// ❌ RUIM — stream não descartado em caso de exceção
var stream = new MemoryStream();
var writer = new StreamWriter(stream);
writer.Write(data); // Exceção aqui = leak!
return stream.ToArray();

// ✅ BOM — using garante Dispose mesmo com exceção
using var stream = new MemoryStream();
await using var writer = new StreamWriter(stream);
await writer.WriteAsync(data);
await writer.FlushAsync();
return stream.ToArray();
```

### IDisposable em services sem escopo correto

```csharp
// ❌ RUIM — Transient IDisposable em Singleton = leak confirmado
services.AddSingleton<IMyService>(sp =>
{
    var dependency = sp.GetRequiredService<IDisposableDependency>(); // Não será descartado!
    return new MyService(dependency);
});

// ✅ BOM — DI gerencia o ciclo de vida correto
services.AddScoped<IDisposableDependency, DisposableDependency>();
services.AddScoped<IMyService, MyService>();
// DI descarta ambos ao fim do scope
```

---

## Logging — dados sensíveis e PII

```csharp
// ❌ RUIM — dados sensíveis no log
_logger.LogInformation("User {Email} logged in with password {Password}",
    user.Email, request.Password); // NUNCA logar senha!

_logger.LogDebug("Processing payment {CardNumber} for {Amount}",
    card.Number, amount); // PAN no log = violação PCI-DSS!

// ✅ BOM — mascarar dados sensíveis
_logger.LogInformation("User {UserId} authenticated successfully", user.Id);
_logger.LogDebug("Processing payment last4={Last4} for amount={Amount}",
    card.Number[^4..], amount);

// ✅ BOM — Enrich sem logar diretamente (usando Serilog/structured logging)
using (_logger.BeginScope(new Dictionary<string, object>
{
    ["UserId"] = userId,
    ["CorrelationId"] = correlationId
}))
{
    _logger.LogInformation("Order confirmed");
    // UserId e CorrelationId aparecem em TODOS os logs deste scope
}
```
