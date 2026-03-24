# Performance — Referência de Problemas e Correções

## LINQ e EF Core

### N+1 Queries — o bug de performance mais comum

```csharp
// ❌ RUIM — 1 query para orders + 1 query POR order para customer = N+1!
var orders = await _context.Orders.ToListAsync(ct);
foreach (var order in orders)
{
    var customer = await _context.Customers.FindAsync(order.CustomerId); // N queries!
    Console.WriteLine($"{customer.Name}: {order.Total}");
}

// ✅ BOM — 1 query com Join
var orders = await _context.Orders
    .Include(o => o.Customer)
    .ToListAsync(ct);

// ✅ MELHOR para leitura — projeção sem carregar entidade completa
var dtos = await _context.Orders
    .Select(o => new OrderSummaryDto(o.Id, o.Customer.Name, o.Total))
    .ToListAsync(ct);
```

### Materialização prematura

```csharp
// ❌ RUIM — carrega TODOS os registros na memória, depois filtra
var active = _context.Products
    .ToList()                              // Executa SQL aqui! 100k registros!
    .Where(p => p.IsActive)               // Filtra em memória
    .OrderBy(p => p.Name)
    .Take(10)
    .ToList();

// ✅ BOM — filtro no SQL
var active = await _context.Products
    .Where(p => p.IsActive)               // WHERE no SQL
    .OrderBy(p => p.Name)
    .Take(10)
    .ToListAsync(ct);
```

### Count em coleção vs banco

```csharp
// ❌ RUIM — carrega dados para contar
var count = _context.Orders.Where(o => o.Active).ToList().Count;
var hasAny = _context.Orders.Where(o => o.Active).ToList().Any();

// ✅ BOM — COUNT(*) no banco
var count = await _context.Orders.Where(o => o.Active).CountAsync(ct);
var hasAny = await _context.Orders.Where(o => o.Active).AnyAsync(ct);
```

### Tracking desnecessário em queries de leitura

```csharp
// ❌ RUIM — EF rastreia entidades que não serão modificadas
var products = await _context.Products.ToListAsync(ct); // Tracked por padrão

// ✅ BOM — AsNoTracking para queries de leitura
var products = await _context.Products
    .AsNoTracking()
    .Where(p => p.IsActive)
    .Select(p => new ProductDto(p.Id, p.Name, p.Price))
    .ToListAsync(ct);
```

---

## Async/Await — pitfalls críticos

### .Result e .Wait() causam deadlock

```csharp
// ❌ RUIM — DEADLOCK em ASP.NET e aplicações com SynchronizationContext
public string GetData()
{
    var result = GetDataAsync().Result;    // DEADLOCK!
    var result2 = GetDataAsync().GetAwaiter().GetResult(); // Também deadlock!
    Task.Run(GetDataAsync).Wait();         // Esconde o problema mas é workaround
    return result;
}

// ✅ BOM — sempre async all the way
public async Task<string> GetDataAsync(CancellationToken ct)
{
    return await _service.GetDataAsync(ct);
}
```

### async void — exceções silenciosas e crashes

```csharp
// ❌ RUIM — exceção mata o processo sem aviso
public async void ProcessPayment(PaymentRequest req)
{
    await _paymentGateway.ChargeAsync(req); // Exceção aqui = crash silencioso!
}

// ✅ BOM — async Task permite await e handling de exceção
public async Task ProcessPaymentAsync(PaymentRequest req, CancellationToken ct)
{
    await _paymentGateway.ChargeAsync(req, ct);
}

// ✅ EXCEÇÃO LEGÍTIMA — event handlers (somente)
private async void OnButtonClick(object sender, EventArgs e)
{
    try { await ProcessAsync(); }
    catch (Exception ex) { await ShowErrorAsync(ex.Message); }
    // Sempre com try/catch em async void!
}
```

### Fire-and-forget incorreto

```csharp
// ❌ RUIM — exceções perdidas, lifecycle não gerenciado
_ = Task.Run(async () => await SendEmailAsync()); // Não aguardado!

// ✅ BOM — IHostedService ou Background Service para tasks de background
public class EmailBackgroundService(IServiceScopeFactory factory)
    : BackgroundService
{
    private readonly Channel<EmailJob> _queue = Channel.CreateBounded<EmailJob>(100);

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var job in _queue.Reader.ReadAllAsync(stoppingToken))
        {
            using var scope = factory.CreateScope();
            var emailSvc = scope.ServiceProvider.GetRequiredService<IEmailService>();
            await emailSvc.SendAsync(job, stoppingToken);
        }
    }
}
```

### ValueTask para hot paths com completion síncrona frequente

```csharp
// Task sempre aloca no heap, mesmo quando retorna sincronamente
public Task<int> GetCachedCountAsync(string key)
{
    if (_cache.TryGetValue(key, out int count))
        return Task.FromResult(count); // Ainda aloca!
    return GetCountFromDbAsync(key);
}

// ✅ ValueTask — zero alocação quando completa sincronamente
public ValueTask<int> GetCachedCountAsync(string key)
{
    if (_cache.TryGetValue(key, out int count))
        return new ValueTask<int>(count); // ZERO alocação!
    return new ValueTask<int>(GetCountFromDbAsync(key));
}
// Use ValueTask quando: resultado frequentemente disponível sincronamente (ex: cache hit)
// Use Task quando: quase sempre assíncrono (ex: DB query)
```

---

## Alocações — reduzir pressão no GC

### ArrayPool para buffers grandes

```csharp
// ❌ RUIM — nova alocação a cada chamada em hot path
public byte[] SerializeToBytes(Order order)
{
    var buffer = new byte[4096]; // Aloca 4KB no heap sempre!
    var written = Serialize(order, buffer);
    return buffer[..written];
}

// ✅ BOM — ArrayPool reutiliza buffers
public byte[] SerializeToBytes(Order order)
{
    var buffer = ArrayPool<byte>.Shared.Rent(4096);
    try
    {
        var written = Serialize(order, buffer);
        return buffer[..written].ToArray(); // Copia apenas o necessário
    }
    finally
    {
        ArrayPool<byte>.Shared.Return(buffer); // Devolve ao pool
    }
}

// ✅ MELHOR — Span<T> para evitar a cópia final
public int SerializeToSpan(Order order, Span<byte> destination)
{
    // Escreve direto no span, sem cópia
    return Serialize(order, destination);
}
```

### StringBuilder vs concatenação

```csharp
// ❌ RUIM — O(n²), cria nova string em cada iteração
string result = "";
foreach (var item in items)
    result += $"{item.Name}: {item.Value}\n"; // N strings intermediárias!

// ✅ BOM — StringBuilder O(n)
var sb = new StringBuilder(items.Count * 30); // Pre-alocar capacidade estimada
foreach (var item in items)
    sb.AppendLine($"{item.Name}: {item.Value}");
return sb.ToString(); // Uma única string no final

// ✅ MELHOR para .NET 6+ — interpolated string handler
// O compilador otimiza automaticamente interpolações com StringBuilder/Span
```

### Span\<T\> para slicing sem alocação

```csharp
// ❌ RUIM — Substring aloca nova string
public bool StartsWithPrefix(string input)
{
    var prefix = input.Substring(0, 4); // Nova alocação!
    return prefix == "GET:" || prefix == "POST";
}

// ✅ BOM — Span sem alocação
public bool StartsWithPrefix(ReadOnlySpan<char> input)
{
    if (input.Length < 4) return false;
    return input[..4] is "GET:" or "POST";
}

// Parsing de CSV sem alocações (hot path)
public static IEnumerable<ReadOnlyMemory<char>> SplitCsv(ReadOnlyMemory<char> input)
{
    while (!input.IsEmpty)
    {
        var idx = input.Span.IndexOf(',');
        if (idx < 0) { yield return input; break; }
        yield return input[..idx];
        input = input[(idx + 1)..];
    }
}
```

---

## Logging — zero allocation em produção

```csharp
// ❌ RUIM — string interpolation = alocação mesmo se log level desabilitado
_logger.LogDebug($"Processing {items.Count} items for order {orderId}");
// Cria a string ANTES de checar se Debug está habilitado!

// 🔶 MELHOR — message template, mas ainda sem otimização máxima
_logger.LogDebug("Processing {ItemCount} items for order {OrderId}",
    items.Count, orderId);
// ILogger faz o formato lazy, mas ainda há boxing de value types

// ✅ MELHOR AINDA — IsEnabled check manual
if (_logger.IsEnabled(LogLevel.Debug))
    _logger.LogDebug("Processing {ItemCount} items for order {OrderId}",
        items.Count, orderId);

// ✅ IDEAL — LoggerMessage source generator (zero allocation, zero boxing)
public static partial class Log
{
    [LoggerMessage(EventId = 100, Level = LogLevel.Debug,
        Message = "Processing {ItemCount} items for order {OrderId}")]
    public static partial void ProcessingOrderItems(
        this ILogger logger, int itemCount, Guid orderId);

    [LoggerMessage(EventId = 500, Level = LogLevel.Error,
        Message = "Failed to process order {OrderId}")]
    public static partial void OrderProcessingFailed(
        this ILogger logger, Guid orderId, Exception ex);
}

// Uso: _logger.ProcessingOrderItems(items.Count, orderId);
```

---

## Estruturas de dados — escolha correta

```csharp
// FrozenDictionary para lookup tables fixas (.NET 8+)
// ~50% mais rápido em TryGetValue vs Dictionary regular
private static readonly FrozenDictionary<string, CountryCode> CountryCodes =
    LoadCountryCodes().ToFrozenDictionary(c => c.Alpha2Code);

// ImmutableArray vs List quando coleção não muda após criação
// ImmutableArray: struct, sem overhead de GC de referência
private static readonly ImmutableArray<string> AllowedRoles =
    ["Admin", "Manager", "Viewer"];

// HashSet para contains frequente
// HashSet.Contains: O(1) vs List.Contains: O(n)
private readonly HashSet<string> _processedIds = new();
if (!_processedIds.Contains(id))
{
    Process(id);
    _processedIds.Add(id);
}

// Channel<T> para producer/consumer assíncrono (melhor que ConcurrentQueue + polling)
var channel = Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait
});
```

---

## HttpClient — evitar socket exhaustion

```csharp
// ❌ RUIM — socket exhaustion após ~1000 requests
public async Task<Product?> GetProductAsync(int id)
{
    using var client = new HttpClient();           // NUNCA faça isso!
    return await client.GetFromJsonAsync<Product>($"/api/products/{id}");
}

// ✅ BOM — IHttpClientFactory gerencia connection pooling
builder.Services.AddHttpClient<IProductApiClient, ProductApiClient>(client =>
{
    client.BaseAddress = new Uri("https://api.example.com");
    client.Timeout = TimeSpan.FromSeconds(30);
    client.DefaultRequestHeaders.Add("Accept", "application/json");
});

// Com resiliência .NET 8+:
builder.Services.AddHttpClient<IProductApiClient, ProductApiClient>()
    .AddStandardResilienceHandler(); // retry + circuit breaker + timeout automáticos

// Com Polly (pre-.NET 8):
builder.Services.AddHttpClient<IProductApiClient, ProductApiClient>()
    .AddTransientHttpErrorPolicy(p =>
        p.WaitAndRetryAsync(3, retry => TimeSpan.FromSeconds(Math.Pow(2, retry))));
```
