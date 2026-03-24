# Template de Relatório de Code Review

## Template completo

```markdown
# 📋 Code Review Report

## Metadata
| Campo | Valor |
|-------|-------|
| **Escopo** | PR #42 — feat: adicionar checkout com Pix |
| **Branch** | feature/pix-checkout → main |
| **Data** | 2025-01-15 |
| **Revisor** | Claude Code Review AI |
| **Stack** | .NET 6.0 \| C# 10 \| Clean Architecture \| EF Core 6 \| MediatR 12.x |
| **Arquivos alterados** | 12 (8 src, 3 tests, 1 migration) |
| **Severidade geral** | 🟠 High — 2 blockers, 3 high, 5 medium |

---

## Sumário Executivo

A implementação do checkout com Pix está funcionalmente correta, mas apresenta 2 pontos
bloqueadores: retry policy configurada para 4xx (pode causar spam ao gateway de pagamento)
e DbContext registrado como Singleton (causará dados stale e crashes em produção).
Há também 3 issues de alta prioridade relacionados a primitive obsession no domínio e
CancellationToken não propagado. Recomendo resolver os blockers antes do merge.

---

## Findings

---

### [RESILIÊNCIA] Retry aplicado em erros 4xx do gateway de pagamento

- **Arquivo:** `src/Infrastructure/PaymentGateway/PixPaymentClient.cs`
- **Linha:** 34-42
- **Severidade:** 🔴 Blocker
- **Tipo:** Resiliência
- **Problema:**
  A retry policy está configurada para tentar novamente em qualquer erro HTTP,
  incluindo 400 Bad Request e 422 Unprocessable Entity. Isso significa que um request
  inválido será enviado 3 vezes ao gateway antes de falhar, potencialmente gerando
  cobranças duplicadas ou bloqueio do IP por rate limit do parceiro.

- **Por que é importante:**
  Erros 4xx representam falhas do cliente, não do servidor. O gateway retorna 422 quando
  os dados de pagamento são inválidos — retry nunca vai consertar dados inválidos.
  Referência: Microsoft Resilience Guidelines, Polly documentation.

- **Código atual:**
  ```csharp
  var retryPolicy = Policy<HttpResponseMessage>
      .HandleResult(r => !r.IsSuccessStatusCode) // ← inclui 4xx!
      .WaitAndRetryAsync(3, _ => TimeSpan.FromSeconds(1));
  ```

- **Código sugerido:**
  ```csharp
  var retryPolicy = Policy<HttpResponseMessage>
      .Handle<HttpRequestException>()
      .OrHandle<TimeoutException>()
      .OrResult(r => (int)r.StatusCode >= 500
                  || r.StatusCode == HttpStatusCode.RequestTimeout
                  || r.StatusCode == HttpStatusCode.TooManyRequests)
      .WaitAndRetryAsync(
          retryCount: 3,
          sleepDurationProvider: attempt =>
              TimeSpan.FromSeconds(Math.Pow(2, attempt))
              + TimeSpan.FromMilliseconds(Random.Shared.Next(0, 500)));
  // Nota: para .NET 8+, considerar AddStandardResilienceHandler()
  ```

- **Impacto do fix:** Elimina risco de cobrança duplicada e bloqueio por rate limit.

---

### [DI] DbContext registrado como Singleton

- **Arquivo:** `src/Api/Program.cs`
- **Linha:** 23
- **Severidade:** 🔴 Blocker
- **Tipo:** Dependency Injection
- **Problema:**
  DbContext registrado como Singleton compartilha o mesmo Change Tracker entre todos
  os requests. Isso causa: dados stale (não vê mudanças de outros requests), change
  tracker crescendo indefinidamente até OOM, e crashes por concorrência em requests
  simultâneos (EF Core não é thread-safe).

- **Por que é importante:**
  DbContext deve ser Scoped (um por HTTP request) — é o padrão do EF Core.
  Referência: EF Core documentation — DbContext Lifetime.

- **Código atual:**
  ```csharp
  builder.Services.AddSingleton<AppDbContext>(sp =>
      new AppDbContext(builder.Configuration.GetConnectionString("Default")));
  ```

- **Código sugerido:**
  ```csharp
  builder.Services.AddDbContext<AppDbContext>(options =>
      options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
  // AddDbContext registra como Scoped por padrão — correto!

  // Adicionar em desenvolvimento para detectar futuros problemas:
  builder.Host.UseDefaultServiceProvider(o =>
  {
      o.ValidateScopes = true;
      o.ValidateOnBuild = true;
  });
  ```

- **Impacto do fix:** Elimina crash por concorrência e dados stale em produção.

---

### [DDD] Primitive Obsession — valor monetário sem Value Object

- **Arquivo:** `src/Domain/Payments/Payment.cs`
- **Linha:** 8-12
- **Severidade:** 🟠 High
- **Tipo:** DDD / Design Pattern
- **Problema:**
  O valor do pagamento é representado como `decimal Amount` sem contexto de moeda.
  Isso permite criar pagamentos sem especificar a moeda, misturar BRL com USD
  silenciosamente, e não há validação de valor negativo ou zero na entidade.

- **Por que é importante:**
  Primitive Obsession é um code smell clássico (Refactoring — Martin Fowler).
  Em domínio financeiro, valor sem moeda é dado inválido por definição.
  Referência: Domain-Driven Design (Evans) — Value Objects.

- **Código atual:**
  ```csharp
  public class Payment
  {
      public decimal Amount { get; set; }     // Qual moeda?
      public string Currency { get; set; }    // Sem validação
      public decimal Fee { get; set; }        // Pode ser negativo?
  }
  ```

- **Código sugerido:**
  ```csharp
  public readonly record struct Money(decimal Amount, string Currency)
  {
      public static Money BRL(decimal amount) => Create(amount, "BRL");
      public static Money USD(decimal amount) => Create(amount, "USD");

      public static Money Create(decimal amount, string currency)
      {
          if (amount < 0)
              throw new DomainException(DomainErrors.Money.NegativeAmount);
          if (string.IsNullOrWhiteSpace(currency) || currency.Length != 3)
              throw new DomainException(DomainErrors.Money.InvalidCurrency);
          return new Money(amount, currency.ToUpperInvariant());
      }

      public Money Add(Money other)
      {
          if (Currency != other.Currency)
              throw new DomainException(DomainErrors.Money.CurrencyMismatch);
          return this with { Amount = Amount + other.Amount };
      }
  }

  public class Payment
  {
      public Money Amount { get; private set; }
      public Money Fee { get; private set; }
      // ...
  }
  ```

- **Impacto do fix:** Elimina possibilidade de dados monetários inválidos no domínio.

---

### [PERFORMANCE] CancellationToken não propagado em operações de banco

- **Arquivo:** `src/Application/Payments/ProcessPixPaymentHandler.cs`
- **Linha:** 28, 35, 42
- **Severidade:** 🟠 High
- **Tipo:** Performance / Resiliência
- **Problema:**
  O `CancellationToken` é recebido como parâmetro mas não é passado para as chamadas
  `FirstOrDefaultAsync()`, `ToListAsync()` e `SaveChangesAsync()`. Isso significa que
  se o usuário cancelar o request (fechar o browser, timeout), as queries no banco
  continuarão executando desnecessariamente, consumindo conexões e recursos.

- **Por que é importante:**
  Em alta carga, queries não-canceláveis acumulam-se e podem esgotar o pool de conexões.
  Referência: Andrew Lock — CancellationToken Best Practices.

- **Código atual:**
  ```csharp
  public async Task<Result> Handle(ProcessPixPaymentCommand cmd, CancellationToken ct)
  {
      var order = await _context.Orders
          .FirstOrDefaultAsync(o => o.Id == cmd.OrderId); // sem ct!
      await _context.SaveChangesAsync();                  // sem ct!
  }
  ```

- **Código sugerido:**
  ```csharp
  public async Task<Result> Handle(ProcessPixPaymentCommand cmd, CancellationToken ct)
  {
      var order = await _context.Orders
          .FirstOrDefaultAsync(o => o.Id == cmd.OrderId, ct); // ct propagado
      if (order is null) return Result.Failure(DomainErrors.Order.NotFound);

      // ... lógica ...

      await _context.SaveChangesAsync(ct); // ct propagado
      return Result.Success();
  }
  ```

- **Impacto do fix:** Reduz consumo de conexões de banco em cenários de cancelamento.

---

### [.NET UPGRADE] MediatR pode requerer licença comercial

- **Arquivo:** `src/Api/Api.csproj`
- **Linha:** 18
- **Severidade:** 🟡 Medium
- **Tipo:** Licença / .NET Ecosystem
- **Problema:**
  O projeto usa `MediatR` versão 12.4.0. A partir da versão 13.0, MediatR não é mais
  open source — passou para licença comercial (Lucky Penny Software, abril 2025).
  Versão 12.x ainda é MIT, mas ao atualizar pacotes pode-se migrar inadvertidamente.

- **Por que é importante:**
  Risco de violação de licença em projetos comerciais ao atualizar dependências.

- **Código atual:**
  ```xml
  <PackageReference Include="MediatR" Version="12.4.0" />
  ```

- **Sugestão:**
  Avaliar migração para alternativa open source:
  ```xml
  <!-- Opção 1: Wolverine (open source, mais features) -->
  <PackageReference Include="WolverineFx.Http" Version="3.x" />

  <!-- Opção 2: martinothamar/Mediator (source generator, high performance, MIT) -->
  <PackageReference Include="Mediator.SourceGenerator" Version="3.x" />
  <PackageReference Include="Mediator.Abstractions" Version="3.x" />
  ```
  Ou congelar versão explicitamente: `Version="[12.4.0]"` (brackets = versão exata).

- **Impacto:** Risco de compliance; zero impacto de performance com versão atual.

---

### [PERFORMANCE] LINQ materialização desnecessária antes de Count

- **Arquivo:** `src/Application/Reports/GetSalesReportHandler.cs`
- **Linha:** 45
- **Severidade:** 🟡 Medium
- **Tipo:** Performance
- **Problema:**
  `ToList()` materializa toda a coleção em memória antes de contar,
  quando `CountAsync()` geraria `SELECT COUNT(*) WHERE...` no banco.
  Com 500k+ orders, isso carrega todos os registros na memória.

- **Código atual:**
  ```csharp
  var total = await _context.Orders
      .Where(o => o.Status == OrderStatus.Completed)
      .ToListAsync(ct)   // Carrega TODOS na memória!
      .Count;            // Conta em memória
  ```

- **Código sugerido:**
  ```csharp
  var total = await _context.Orders
      .Where(o => o.Status == OrderStatus.Completed)
      .CountAsync(ct);   // SELECT COUNT(*) WHERE Status = 'Completed'
  ```

- **Impacto do fix:** De O(n) memória para O(1) memória; reduz tempo de ~2s para ~50ms.

---

### [PATTERN] Logging com string interpolation em hot path

- **Arquivo:** `src/Infrastructure/PaymentGateway/PixPaymentClient.cs`
- **Linha:** 67, 73, 89
- **Severidade:** 🟢 Low
- **Tipo:** Performance / Logging
- **Problema:**
  String interpolation com ILogger cria a string ANTES de verificar se o log level
  está habilitado. Em produção com Debug desabilitado, ainda há custo de formatação.

- **Código atual:**
  ```csharp
  _logger.LogDebug($"Processing Pix payment for order {orderId}, amount {amount}");
  _logger.LogDebug($"Gateway response: {JsonSerializer.Serialize(response)}");
  ```

- **Código sugerido:**
  ```csharp
  // Opção 1: Message template (lazy formatting)
  _logger.LogDebug("Processing Pix payment for order {OrderId}, amount {Amount}",
      orderId, amount);

  // Opção 2: Source generator para zero allocation (recomendado para hot path)
  public static partial class PixLog
  {
      [LoggerMessage(Level = LogLevel.Debug,
          Message = "Processing Pix payment for order {OrderId}, amount {Amount}")]
      public static partial void ProcessingPayment(
          this ILogger logger, Guid orderId, decimal amount);
  }
  _logger.ProcessingPayment(orderId, amount);
  ```

- **Impacto do fix:** Elimina alocação de string quando Debug está desabilitado.

---

### [TESTES] Lógica de cálculo de taxa Pix sem testes unitários

- **Arquivo:** `src/Domain/Payments/PixFeeCalculator.cs`
- **Linha:** 1-45 (arquivo inteiro)
- **Severidade:** 🟠 High
- **Tipo:** Testes
- **Problema:**
  `PixFeeCalculator` contém lógica de negócio de cálculo de taxa (0.99% para pessoa
  física, 1.49% para jurídica, isenção para valores < R$10) sem nenhum teste unitário.
  Erro nesta lógica impacta diretamente o faturamento.

- **Por que é importante:**
  Cálculos financeiros são o tipo de lógica que mais requer testes (Clean Code — Robert
  Martin: toda lógica de negócio deve ter testes automatizados).

- **Sugestão:**
  ```csharp
  public class PixFeeCalculatorTests
  {
      [Theory]
      [InlineData(100m, CustomerType.Individual, 0.99)]    // 0.99%
      [InlineData(100m, CustomerType.Business, 1.49)]      // 1.49%
      [InlineData(9.99m, CustomerType.Individual, 0)]      // Isento < R$10
      [InlineData(1000m, CustomerType.Business, 14.9)]     // 1.49% de 1000
      public void Calculate_ReturnsCorrectFee(
          decimal amount, CustomerType type, decimal expectedFee)
      {
          var calculator = new PixFeeCalculator();
          var fee = calculator.Calculate(Money.BRL(amount), type);
          fee.Amount.Should().Be(expectedFee);
      }

      [Fact]
      public void Calculate_NegativeAmount_ThrowsDomainException()
      {
          var calculator = new PixFeeCalculator();
          var act = () => calculator.Calculate(Money.BRL(-1), CustomerType.Individual);
          act.Should().Throw<DomainException>()
              .WithMessage("*negative*");
      }
  }
  ```

---

## Oportunidades de Upgrade .NET

> Projeto em .NET 6 — EOL novembro 2024. Recomendado migrar para .NET 8 LTS.

| Feature | Benefício | Esforço |
|---------|-----------|---------|
| Migrar para .NET 8 | PGO por padrão (+15-30% geral), LINQ 50%+ mais rápido | Médio |
| `FrozenDictionary` para tabelas de taxa | ~50% mais rápido em lookups | Baixo |
| `LoggerMessage` source generator | Zero-allocation logging | Baixo |
| Primary constructors | Menos boilerplate em handlers | Baixo |
| `DateOnly` para datas de vencimento | Sem bugs de timezone | Baixo |

---

## Padrões do projeto quebrados

1. **Nomenclatura:** `ProcessPixPaymentHandler.cs` usa `Handle()` — projeto usa `HandleAsync()` nos demais handlers. Padronizar.
2. **Retorno:** Outros handlers retornam `Result<T>` via `ICommandHandler<,>`, mas este retorna `Unit` diretamente. Inconsistente.
3. **Testes:** Demais features têm 80%+ de cobertura em regras de negócio; esta feature tem 0% em `PixFeeCalculator`.

---

## Pontos positivos ✅

- Migration do EF Core bem estruturada com índice na coluna `pix_key`
- `PixPaymentClient` corretamente injetado via `IHttpClientFactory`
- DTOs usando `record` com validação via FluentValidation
- Endpoint usando `TypedResults` (bom!) — retorno tipado e documentado no Swagger
- Cobertura de teste para o caso de idempotência (evitar cobrança dupla)

---

## Checklist de aprovação

- [ ] 🔴 Retry policy corrigida para não incluir 4xx
- [ ] 🔴 DbContext registrado como Scoped
- [ ] 🟠 Money Value Object adicionado ao domínio
- [ ] 🟠 CancellationToken propagado nas queries
- [ ] 🟠 Testes unitários para PixFeeCalculator
- [ ] 🟡 MediatR version pinned (ou migração avaliada)
- [ ] 🟡 Logging com message templates (não interpolação)
- [ ] 🟢 CountAsync em vez de ToList().Count
```
