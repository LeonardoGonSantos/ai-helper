# Versões do .NET — Features, Performance e Upgrade Guide

## Detecção rápida de versão

```csharp
// C# 9 / .NET 5 — Records, init, pattern matching melhorado
public record PersonDto(string Name, int Age);
if (obj is not null) { }

// C# 10 / .NET 6 — global using, file-scoped namespace, DateOnly
global using System.Collections.Generic;
namespace MyApp; // sem chaves

// C# 11 / .NET 7 — required members, raw string literals, generic math
public required string Name { get; init; }
var json = """{"name": "test"}""";

// C# 12 / .NET 8 — primary constructors, collection expressions
public class Service(ILogger<Service> logger) { }
int[] nums = [1, 2, 3, ..otherNums];

// C# 13 / .NET 9 — Lock type, params collections, partial properties
private readonly Lock _lock = new();

// C# 14 / .NET 10 — field keyword, null-conditional assignment, extension blocks
public string Name { get => field; set => field = value?.Trim() ?? ""; }
```

---

## .NET 5 — Baseline (EOL maio 2022)

**Status:** ⛔ EOL — sugerir upgrade urgente para .NET 8 LTS.

### Features C# 9 disponíveis
- `record` types (imutáveis por padrão)
- `init`-only setters
- Pattern matching melhorado: `not null`, `or`, `and`
- Top-level statements

### Problemas críticos para sinalizar
- Sem `DateOnly`/`TimeOnly` → uso de `DateTime` para datas-apenas
- `MinBy`/`MaxBy` ausentes → `OrderBy().First()` = O(n log n)
- `Chunk()` ausente → paginação manual verbosa
- Runtime sem PGO automático → performance subótima

```csharp
// ❌ .NET 5 — padrão comum a sinalizar
var youngest = people.OrderBy(p => p.Age).First(); // O(n log n)!
var batches = items.Select((x, i) => (x, i))
    .GroupBy(t => t.i / 100, t => t.x)
    .Select(g => g.ToList()).ToList(); // verboso e lento

// ✅ Sugestão de upgrade para .NET 6+
var youngest = people.MinBy(p => p.Age);  // O(n)
var batches = items.Chunk(100);            // limpo e eficiente
```

---

## .NET 6 — LTS (EOL novembro 2024)

**Status:** ⚠️ EOL — migrar para .NET 8 LTS.

### Features chave
- **Minimal APIs** — sem Controllers, alto throughput
- **DateOnly / TimeOnly** — sem zona horária acidental
- **LINQ** — `MinBy`, `MaxBy`, `Chunk`, `DistinctBy`, `ExceptBy`
- **Hot Reload** em desenvolvimento

```csharp
// DateOnly para campos que são apenas data
public record EmployeeDto(string Name, DateOnly BirthDate, TimeOnly StartTime);

// Minimal API
app.MapGet("/products/{id}", async (int id, AppDbContext db, CancellationToken ct)
    => await db.Products.FindAsync([id], ct) is Product p
        ? Results.Ok(p)
        : Results.NotFound());
```

### O que sinalizar no .NET 6
- `DateTime` para datas sem hora → sugerir `DateOnly`
- `OrderBy(x => x.Prop).First()` → sugerir `MinBy`
- Loop manual para chunking → sugerir `Chunk()`
- Controllers novos em apps simples → sugerir Minimal API

---

## .NET 7 (EOL maio 2024)

**Status:** ⚠️ EOL — migrar para .NET 8 LTS.

### Features chave
- **`required` members** — validação de inicialização em compile-time
- **Generic Math** (`INumber<T>`, `IAdditionOperators<T,T,T>`)
- **Rate Limiting middleware** built-in
- **Output Caching** middleware
- Regex source generators
- `nameof` em atributos

```csharp
// required garante inicialização correta
public class CreateUserDto
{
    public required string Name { get; init; }
    public required string Email { get; init; }
    // Sem required = erro de compilação ao instanciar sem os campos
}

// Rate limiting built-in (sem AspNetCoreRateLimit)
builder.Services.AddRateLimiter(options =>
    options.AddFixedWindowLimiter("api", opt =>
    {
        opt.PermitLimit = 100;
        opt.Window = TimeSpan.FromMinutes(1);
    }));

app.UseRateLimiter();
app.MapGet("/", () => "Hello").RequireRateLimiting("api");
```

### O que sinalizar no .NET 7
- DTOs com propriedades opcionais onde `required` faria mais sentido
- Uso de pacotes externos de rate limiting → sugerir built-in

---

## .NET 8 — LTS atual (suporte até novembro 2026)

**Status:** ✅ Versão LTS recomendada para novos projetos.

### Features chave de performance
- **Dynamic PGO por padrão** — até 15% de ganho geral sem mudanças
- **FrozenDictionary / FrozenHashSet** — ~50% mais rápido em lookups read-only
- **SearchValues\<T\>** — busca vetorizada (SIMD) em strings
- **LoggerMessage source generator** — zero-allocation logging
- **Collection expressions** — `[1, 2, 3]`, `[..a, ..b]`
- **Primary constructors**
- **`IHostedService`** melhorias

```csharp
// FrozenDictionary para lookup tables (configs, mapeamentos, etc.)
private static readonly FrozenDictionary<string, int> StatusCodes =
    new Dictionary<string, int>
    {
        ["pending"] = 1, ["active"] = 2, ["cancelled"] = 3
    }.ToFrozenDictionary();
// ~50% mais rápido que Dictionary normal em TryGetValue

// SearchValues — busca em string sem regex overhead
private static readonly SearchValues<char> Separators =
    SearchValues.Create(" \t\n,;");

public static int CountWords(ReadOnlySpan<char> text)
{
    int count = 0;
    while (!text.IsEmpty)
    {
        int idx = text.IndexOfAny(Separators);
        if (idx < 0) { count++; break; }
        if (idx > 0) count++;
        text = text[(idx + 1)..];
    }
    return count;
}

// LoggerMessage source generator — zero allocation
public static partial class Log
{
    [LoggerMessage(Level = LogLevel.Information,
        Message = "Processing order {OrderId} for customer {CustomerId}")]
    public static partial void ProcessingOrder(
        this ILogger logger, Guid orderId, Guid customerId);
}
// Uso: _logger.ProcessingOrder(orderId, customerId);

// Primary constructors
public class OrderService(
    IOrderRepository repo,
    ILogger<OrderService> logger,
    IUnitOfWork unitOfWork)
{
    public async Task<Result<OrderId>> CreateAsync(
        CreateOrderCommand cmd, CancellationToken ct)
    {
        logger.ProcessingOrder(Guid.NewGuid(), cmd.CustomerId);
        // ...
    }
}
```

### O que sinalizar em projetos .NET 8 (melhorias não aproveitadas)
- `Dictionary` read-only inicializado 1x → sugerir `FrozenDictionary`
- `ILogger.LogInformation($"...")` → sugerir `LoggerMessage` source generator
- `class` para DTOs → sugerir `record`
- Constructor com muitos campos → sugerir primary constructors

---

## .NET 9 (suporte até maio 2026)

**Status:** ✅ STS — válido, mas .NET 10 LTS será lançado novembro 2025.

### Features chave
- **`CountBy` e `AggregateBy`** no LINQ
- **`Task.WhenEach`** — processa tasks conforme completam
- **HybridCache** — L1 (memória) + L2 (distribuído) com stampede protection
- **`Lock` type dedicado** — mais eficiente que `object` com `lock`
- **`params` em coleções** — `params ReadOnlySpan<T>`
- **`SearchValues` melhorado** com suporte a strings

```csharp
// CountBy — substitui GroupBy + Count
var ordersByStatus = orders.CountBy(o => o.Status);
// Em vez de: orders.GroupBy(o => o.Status).ToDictionary(g => g.Key, g => g.Count())

// AggregateBy — substitui GroupBy + aggregate
var totalByCategory = products.AggregateBy(
    p => p.Category,
    0m,
    (total, p) => total + p.Price);

// Task.WhenEach — processa conforme chegam (não todos de uma vez)
await foreach (var task in Task.WhenEach(t1, t2, t3, t4))
{
    var result = await task;
    ProcessResult(result); // Processa assim que cada task conclui
}

// HybridCache — substitui padrão manual de L1+L2 cache
public class ProductService(HybridCache cache, AppDbContext db)
{
    public async Task<ProductDto?> GetAsync(int id, CancellationToken ct)
        => await cache.GetOrCreateAsync(
            $"product:{id}",
            async token => await db.Products
                .Where(p => p.Id == id)
                .Select(p => new ProductDto(p.Id, p.Name, p.Price))
                .FirstOrDefaultAsync(token),
            cancellationToken: ct);
}
```

---

## .NET 10 — LTS (novembro 2025)

**Status:** ✅ Próximo LTS — planejar migração de .NET 8.

### Features chave
- **`field` keyword** — acesso ao backing field sem campo explícito
- **Null-conditional assignment** — `obj?.Property = value`
- **Extension blocks** com propriedades e membros estáticos
- **Nullable reference types** melhorado
- **LINQ** — `CountBy`/`AggregateBy` (retroportado do v9)
- **JIT** — escape analysis avançado, mais alocações stack

```csharp
// field keyword — propriedade com lógica sem backing field extra
public class User
{
    public string Name
    {
        get => field;
        set => field = value?.Trim() ?? throw new ArgumentNullException(nameof(value));
    }

    public string Email
    {
        get => field;
        set => field = value?.ToLowerInvariant()
            ?? throw new ArgumentNullException(nameof(value));
    }
}

// Extension blocks — propriedades de extensão reais!
extension(string s)
{
    public bool IsNullOrEmpty => string.IsNullOrEmpty(s);
    public string Capitalize() => string.IsNullOrEmpty(s)
        ? s : char.ToUpperInvariant(s[0]) + s[1..];
}

// Null-conditional assignment
config?.Timeout = TimeSpan.FromSeconds(30);
```

### Performance .NET 10
- Escape analysis avançado: mais objetos alocados no stack → menos pressão no GC
- Thread pool scheduling: até **5000× mais rápido** em workloads específicos vs .NET 9
- JSON serialization: melhorias adicionais em throughput

---

## Matriz de upgrade — o que sugerir

```
Versão atual → Ação recomendada

.NET Framework 4.x  → Migrar para .NET 8 LTS (URGENTE — sem novas features, segurança limitada)
.NET 5              → Migrar para .NET 8 LTS (EOL, 2 anos sem suporte)
.NET 6              → Migrar para .NET 8 LTS (EOL nov/2024)
.NET 7              → Migrar para .NET 8 LTS (EOL maio/2024)
.NET 8              → Continuar (LTS até nov/2026), planejar migração .NET 10 para 2026
.NET 9              → OK, planejar .NET 10 quando GA
.NET 10             → Atual LTS ✅
```

### Argumentos de performance para justificar upgrade

| De | Para | Ganho estimado |
|---|---|---|
| .NET 6 | .NET 8 | ~30-40% em operações web típicas |
| .NET 7 | .NET 8 | ~15-20% (PGO habilitado por padrão) |
| .NET 6 | .NET 8 | LINQ: 50%+ em muitas operações |
| .NET 8 | .NET 10 | GC: redução de 70-90% de pressão em alguns workloads |
| Qualquer | .NET 8+ | FrozenDictionary lookups: ~50% |
| Qualquer | .NET 8+ | SearchValues (SIMD): ~3-10× vs manual |
