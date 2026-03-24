---
name: feature-build
description: >-
  Execute a .NET C# feature implementation plan step-by-step using TDD
  (Green-Refactor phases). Reads the plan.md and spec.md created by
  init-feature, implements one task at a time, runs dotnet build and dotnet
  test after each task, commits atomically, and only marks the feature complete
  when 100% of tests pass and all acceptance criteria are met. Finishes with a
  re-analysis confirming the original plan was fully satisfied. Use this skill
  whenever the user says "build feature", "implementar", "executar o plano",
  "codificar feature", "continuar feature", "implement", or references a
  plan.md. Never skip the build/test cycle — a task is only done when it
  compiles and tests are green.
---

# Feature Build

## Overview
Execute the implementation plan created by `init-feature`. One task at a time,
following TDD Green-Refactor. Every task must compile and pass tests before
moving to the next. The feature is only complete when 100% of tests pass, the
build is clean in Release mode, and every acceptance criterion is verified.

---

## Pre-Flight

1. Leia: `docs/design/FEAT-{name}/plan.md` — carregue a lista de tarefas
2. Leia: `docs/design/FEAT-{name}/spec.md` — carregue os critérios de aceite
3. Leia: `docs/design/FEAT-{name}/sdd.md` — entenda o design e decisões
4. Verifique os testes em estado Red: `dotnet test --filter "FullyQualifiedName~{FeatureName}"`
5. Confirme branch limpa: `git status`
6. Detecte o result pattern e DI registration style lendo uma feature existente

Se algum pré-flight falhar, PARE e informe o usuário.

---

## Implementation Loop

Para CADA tarefa no `plan.md` (em ordem estrita):

### Passo 1: ANUNCIAR
"Implementando Tarefa N: {descrição}"

### Passo 2: IMPLEMENTAR
Escreva o mínimo de código para cumprir a tarefa, seguindo os padrões do projeto.

**Command + Handler (CQRS/MediatR):**
```csharp
public record Create{Entity}Command(string Name, decimal Price)
    : IRequest<ErrorOr<Guid>>;  // adapte ao result pattern do projeto

public class Create{Entity}CommandHandler
    : IRequestHandler<Create{Entity}Command, ErrorOr<Guid>>
{
    private readonly I{Entity}Repository _repository;
    private readonly IUnitOfWork _unitOfWork;
    private readonly IPublisher _publisher;  // remova se não usar domain events

    public Create{Entity}CommandHandler(
        I{Entity}Repository repository,
        IUnitOfWork unitOfWork,
        IPublisher publisher)
    {
        _repository = repository;
        _unitOfWork = unitOfWork;
        _publisher = publisher;
    }

    public async Task<ErrorOr<Guid>> Handle(
        Create{Entity}Command request, CancellationToken ct)
    {
        var entity = {Entity}.Create(request.Name, request.Price);
        if (entity.IsError) return entity.Errors;

        await _repository.AddAsync(entity.Value, ct);
        await _unitOfWork.SaveChangesAsync(ct);

        await _publisher.Publish(
            new {Entity}CreatedEvent(entity.Value.Id), ct);

        return entity.Value.Id;
    }
}
```

**Pipeline Behavior (ValidationBehavior — verificar se já existe):**
```csharp
// Só crie se não existir no projeto
public class ValidationBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;
    // ...
}
```

**Endpoint (Minimal API):**
```csharp
public static class {Entity}Endpoints
{
    public static void Map{Entity}Endpoints(this WebApplication app)
    {
        var group = app.MapGroup("/api/{entities}")
            .WithTags("{Entities}")
            .RequireAuthorization();

        group.MapPost("/", async (
            Create{Entity}Command command,
            IMediator mediator,
            CancellationToken ct) =>
        {
            var result = await mediator.Send(command, ct);
            return result.Match(
                id => Results.Created($"/api/{entities}/{id}", new { id }),
                errors => Results.Problem(errors.ToProblemDetails()));
        })
        .WithName("Create{Entity}")
        .WithSummary("Cria um novo {entity}")
        .Produces<object>(StatusCodes.Status201Created)
        .ProducesValidationProblem()
        .Produces(StatusCodes.Status401Unauthorized);
    }
}
```

**DI Registration (adapte ao padrão do projeto):**
```csharp
// Em DependencyInjection.cs ou ServiceCollectionExtensions.cs
services.AddScoped<I{Entity}Repository, {Entity}Repository>();
// Validators: services.AddValidatorsFromAssemblyContaining<Create{Entity}CommandValidator>();
// Endpoints: app.Map{Entity}Endpoints();
```

**EF Core Configuration:**
```csharp
public class {Entity}Configuration : IEntityTypeConfiguration<{Entity}>
{
    public void Configure(EntityTypeBuilder<{Entity}> builder)
    {
        builder.ToTable("{entities}");
        builder.HasKey(x => x.Id);
        builder.Property(x => x.Name).HasMaxLength(200).IsRequired();

        // Se multitenancy com global query filter:
        builder.HasQueryFilter(x => x.TenantId == EF.Property<Guid>(x, "_tenantId"));
    }
}
```

**HTTP Client com Polly (se chamada externa):**
```csharp
services.AddHttpClient<I{External}Client, {External}Client>(client =>
{
    client.BaseAddress = new Uri(config["{External}:BaseUrl"]!);
})
.AddResiliencePipeline("default", builder =>
{
    builder
        .AddRetry(new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            Delay = TimeSpan.FromSeconds(1),
            BackoffType = DelayBackoffType.Exponential,
        })
        .AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
        {
            SamplingDuration = TimeSpan.FromSeconds(10),
            MinimumThroughput = 5,
            FailureRatio = 0.5,
        })
        .AddTimeout(TimeSpan.FromSeconds(5));
});
```

### Passo 3: BUILD
```bash
dotnet build
```
- Se falhar: corrija imediatamente. Máximo 3 tentativas.
- Se ainda falhar: PARE e informe o usuário com o erro completo.

### Passo 4: TEST
```bash
dotnet test
```
- Registre quais testes mudaram Red → Green
- Se houver regressões: corrija antes de prosseguir. Máximo 3 tentativas.

### Passo 5: REFACTOR (se necessário)
- Extraia duplicação, melhore nomes, garanta responsabilidade única
- Execute `dotnet test` novamente — deve continuar passando

### Passo 6: COMMIT ATÔMICO
```bash
git add -A
git commit -m "feat({scope}): {descrição da tarefa no imperativo}"
```
- Conventional Commits. SEM `Co-Authored-By`. SEM atribuição de IA.

### Passo 7: ATUALIZAR PLANO
```
- [x] Tarefa N: {descrição} ✅
```

**Repita para a próxima tarefa.**

---

## Ponto de Atenção: Migrations

Quando a tarefa envolver mudança de schema:
1. Crie a migration: `dotnet ef migrations add {MigrationName} --project src/Infrastructure --startup-project src/Api`
2. Revise o arquivo gerado — confirme que está correto
3. **PERGUNTE** antes de aplicar: "Posso aplicar a migration `{MigrationName}` agora?"
4. Só aplique após confirmação: `dotnet ef database update`

---

## Ponto de Atenção: Testes de Integração (Testcontainers)

Após os testes unitários passarem, se a feature tiver endpoints:

```csharp
public class {Feature}IntegrationTests
    : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public {Feature}IntegrationTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureTestServices(services =>
            {
                // Substituir DbContext para usar Testcontainers
                services.RemoveAll<DbContextOptions<AppDbContext>>();
                services.AddDbContext<AppDbContext>(options =>
                    options.UseNpgsql(TestDatabase.ConnectionString));
            });
        }).CreateClient();
    }

    [Fact]
    public async Task POST_{Entity}_ValidRequest_Returns201()
    {
        var request = new { /* campos válidos */ };
        var response = await _client.PostAsJsonAsync("/api/{entities}", request);
        response.StatusCode.Should().Be(HttpStatusCode.Created);
    }

    [Fact]
    public async Task POST_{Entity}_InvalidRequest_Returns422WithErrors()
    {
        var request = new { /* campos inválidos */ };
        var response = await _client.PostAsJsonAsync("/api/{entities}", request);
        response.StatusCode.Should().Be(HttpStatusCode.UnprocessableEntity);
        var problem = await response.Content.ReadFromJsonAsync<ValidationProblemDetails>();
        problem!.Errors.Should().ContainKey("{campo}");
    }
}
```

---

## Verificação Periódica (a cada 3 tarefas)
```bash
dotnet build --configuration Release
dotnet test --verbosity normal
dotnet format --verify-no-changes
```

---

## Critérios de Conclusão
- [ ] Todas as tarefas em `plan.md` marcadas como `[x] ✅`
- [ ] `dotnet build --configuration Release` → 0 erros, 0 warnings
- [ ] `dotnet test` → 100% passando (unitários + integração)
- [ ] `dotnet format --verify-no-changes` → limpo
- [ ] Nenhum `// TODO` ou `// HACK` no código novo
- [ ] Todos os CAs do `spec.md` cobertos por testes passando
- [ ] DI registration verificado (todos os serviços resolvem sem erro na startup)
- [ ] Migration aplicada e schema atualizado (se aplicável)

---

## Re-análise Final (obrigatória)

1. Releia todos os critérios de aceite do `spec.md`
2. Para cada CA, identifique o teste que o cobre e confirme que está passando
3. Verifique se a implementação corresponde ao design no `sdd.md`
4. Liste qualquer desvio do plano e justifique

```
## Feature Build Concluída ✅

### Cobertura dos Critérios de Aceite
| CA    | Descrição       | Teste que cobre          | Status |
|-------|-----------------|--------------------------|--------|
| CA-01 | {descrição}     | {TestClass.MethodName}   | ✅     |

### Resumo
- Tarefas: X/X concluídas
- Testes: Y passando (Z unitários + W integração)
- Arquivos criados: [lista]
- Migrations: [lista ou "nenhuma"]
- Desvios do plano: {nenhum / lista justificada}

### Próximo passo: finish-feature
```

---

## Guardrails
- UMA tarefa por vez — nunca agrupe múltiplas tarefas em um commit
- NÃO pule o ciclo build/test para nenhuma tarefa
- NÃO modifique testes existentes sem aprovação explícita do usuário
- NÃO aplique migrations sem confirmação do usuário
- Se travado por mais de 3 tentativas: PARE e explique o bloqueio
- Máximo 5 commits consecutivos sem verificação completa
- Se o plano precisar de ajustes: atualize `plan.md` e informe antes de continuar
- Sempre adapte os padrões de código ao result pattern detectado no projeto
