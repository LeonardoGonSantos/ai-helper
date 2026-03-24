---
name: code-review
description: >
  Executa um code review profundo e estruturado de projetos .NET C# (versões 5 a 10),
  cobrindo PR, commits ou changelogs. Identifica automaticamente a stack e versão do .NET,
  analisa design patterns ausentes ou mal aplicados (Strategy, Factory, Builder, Decorator,
  Repository, CQRS), detecta violações de Clean Architecture, DDD (Value Objects, Anemic
  Domain Model, Aggregates), falhas de resiliência (retry em 4xx, falta de circuit breaker,
  CancellationToken ausente), problemas de async/await, memory leaks, logging incorreto e
  perdas de performance. Gera relatório Markdown estruturado com arquivo, linha, exemplo de
  código corrigido e justificativa técnica. Use este skill SEMPRE que o usuário mencionar
  "review", "revisar PR", "revisar commit", "revisar código", "code review", "analisar PR",
  "analisar mudanças", "checar qualidade", "revisar alterações" ou pedir análise de qualquer
  arquivo ou diff .NET C#.
---

# Code Review — .NET C# (v5 → v10)

> Referências detalhadas por categoria em `references/`. Leia o arquivo correspondente
> em cada step antes de emitir diagnósticos — não assuma valores de memória.

---

## STEP 0 — Escolher escopo do review

**Sempre pergunte antes de qualquer análise:**

```
Qual é o escopo deste review?
  1. PR (Pull Request) — diff completo entre branches
  2. Commit(s) específico(s) — hash ou range
  3. Changelog / release notes — lista de mudanças textuais
  4. Arquivo(s) avulso(s) — análise direta de código

Informe o número ou cole o conteúdo diretamente.
```

Se o usuário colar código diretamente, inferir escopo como "Arquivo(s) avulso(s)" e prosseguir.

---

## STEP 1 — Ingestão do material de review

### Para PR / Diff
- Receber o diff (texto `git diff`, URL do PR ou arquivos modificados)
- Listar todos os arquivos alterados e classificar por tipo:
  - **Domain** (Entities, Value Objects, Aggregates, Domain Events)
  - **Application** (Commands, Queries, Handlers, DTOs, Validators)
  - **Infrastructure** (DbContext, Repositories, HTTP clients, Migrations)
  - **Presentation** (Controllers, Endpoints, Middleware)
  - **Tests** (Unit, Integration, E2E)
  - **Config** (`.csproj`, `appsettings`, `Program.cs`, `Startup.cs`)

### Para Commit
- Ingerir mensagem do commit + diff associado
- Checar se a mensagem segue Conventional Commits:
  - `feat:`, `fix:`, `refactor:`, `perf:`, `test:`, `chore:`, `docs:`
  - ⚠️ Mensagens vagas ("fix", "wip", "changes") → sinalizar

### Para Changelog / Release Notes
- Parsear itens listados e inferir quais arquivos/camadas foram afetados
- Sinalizar itens sem cobertura de testes mencionada

---

## STEP 2 — Identificar stack e versão do .NET

Detectar a versão do .NET a partir de:
1. `<TargetFramework>` no `.csproj` (`net5.0`, `net6.0`, `net7.0`, `net8.0`, `net9.0`, `net10.0`)
2. Sintaxe C# usada (primary constructors → C#12/.NET8+; records → C#9/.NET5+)
3. APIs utilizadas (`DateOnly` → .NET6+; `FrozenDictionary` → .NET8+)

**Leia `references/dotnet-versions.md` para a análise completa desta etapa.**

### Resultado esperado desta etapa

```
Stack detectada:
- .NET: 6.0 (C# 10)
- Arquitetura: Clean Architecture + CQRS
- Mediator: MediatR 12.x  ← checar licença se ≥ v13
- ORM: EF Core 6
- Testes: xUnit + Moq
- DI: Microsoft.Extensions.DependencyInjection

Oportunidades de upgrade identificadas: [lista aqui]
```

---

## STEP 3 — Identificar padrões do projeto

Antes de criticar, mapear o que o projeto JÁ ADOTA para manter consistência.

**Checar:**
- [ ] Convenção de nomenclatura: PascalCase para types, camelCase para locals/params, `_camelCase` para fields privados
- [ ] Sufixos: `Service`, `Repository`, `Handler`, `Query`, `Command`, `Dto`, `Request`, `Response`, `Validator`, `Factory`, `Strategy`
- [ ] Organização de pastas: por layer vs por feature (Vertical Slice)
- [ ] Padrão de retorno: `Result<T>`, exceções, `IActionResult`, `TypedResults`
- [ ] Error handling: global middleware? `ProblemDetails`? Exceções de domínio?
- [ ] Async convention: `Async` suffix sempre? `CancellationToken` padrão?
- [ ] Testes: nomeação `Method_Scenario_Expected`, Given-When-Then, `[Fact]` vs `[Theory]`

**Quebras de padrão encontradas são HIGH PRIORITY** — inconsistência é pior que convenção ruim.

---

## STEP 4 — Análise de Design Patterns

**Leia `references/patterns.md` para exemplos completos de cada pattern.**

### 4A — Detectar oportunidades de Strategy

**Trigger:** `if/else if` ou `switch` com **3+ branches** contendo lógica de negócio distinta.

Sinalizar quando:
- Cada branch instancia objeto diferente ou chama método diferente
- Novos casos exigiriam modificar o método (viola OCP)
- O switch está em Service, Handler ou Domain

Não sinalizar quando:
- Switch simples de mapeamento/lookup (1-2 linhas por case)
- Expressão de pattern matching para parsing/conversão

### 4B — Detectar oportunidades de Factory / Abstract Factory

**Trigger:** `new ConcreteClass()` em lógica de negócio, construtores complexos repetidos, criação condicional de objetos.

Sinalizar quando:
- `new` em Service ou Handler (não em Composition Root / DI)
- Criação de objetos com 4+ parâmetros repetida em vários lugares
- Lógica condicional de criação (`if x new A() else new B()`)

### 4C — Detectar oportunidades de Builder

**Trigger:** construtores com **5+ parâmetros**, objetos complexos construídos passo a passo com setters, test data sem builder.

```csharp
// ❌ RUIM: construtor com 7 parâmetros
var order = new Order(customerId, productId, qty, price, currency, address, "PENDING");

// ✅ SUGERIR: Builder / object initialization com record
var order = Order.Create(customerId, product, qty, address);
// OU test builder para testes
var order = new OrderBuilder().ForCustomer(customerId).WithItems(1).Build();
```

### 4D — Detectar oportunidades de Decorator

**Trigger:** mesmo método com logging + caching + retry + validação **todos juntos** no mesmo service.

### 4E — Detectar oportunidades de Mediator / CQRS sem biblioteca

**Trigger:** Services com 5+ responsabilidades distintas, ou MediatR ≥ v13 sem licença.

⚠️ **ALERTA OBRIGATÓRIO se MediatR for detectado:**
> Verificar versão. MediatR ≥ 13.0 não é mais open source (licença comercial desde abril 2025 via Lucky Penny Software). Alternativas gratuitas: **Wolverine** (open source, convention-based), **martinothamar/Mediator** (source generator, high performance, MIT), implementação própria com `ICommandHandler<TCommand, TResult>`.

### 4F — Detectar oportunidades de Specification Pattern

**Trigger:** múltiplos métodos no repository com variações de filtro (`GetActiveUsers`, `GetAdminUsers`, `GetUsersByCity`...).

---

## STEP 5 — Análise de DDD

**Leia `references/ddd.md` para exemplos completos.**

### 5A — Primitive Obsession → sugerir Value Objects

Buscar no código:
- `string` representando: Email, CPF, CNPJ, telefone, URL, código de produto, moeda, status
- `decimal` sem contexto de moeda
- `int` para IDs tipados (confusão entre `orderId` e `customerId`)
- Grupo de primitivos que sempre viajam juntos (rua + cidade + CEP = `Address`)

### 5B — Anemic Domain Model

Sinalizar quando entidades têm:
- Apenas `{ get; set; }` sem métodos de negócio
- Coleções públicas mutáveis (`List<T>` ou `ICollection<T>` públicos)
- Construtor sem parâmetros permitindo estado inválido
- Regras de negócio em Services operando em "data bags"

### 5C — Aggregate boundaries

- Coleções devem ser `IReadOnlyCollection<T>` externamente
- Referências entre Aggregates apenas por ID (não navegação direta)
- Um repository por Aggregate Root
- Todas as mutações via métodos com nomes do Ubiquitous Language

### 5D — Regras de negócio fora do domínio

Sinalizar quando Controllers ou Handlers contêm:
- Validação de estado de domínio (`if (order.Status != "pending")`)
- Cálculos de negócio (desconto, imposto, totais)
- Mutações diretas de propriedade (`order.Status = "confirmed"`)

---

## STEP 6 — Análise de Clean Architecture

**Leia `references/architecture.md` para exemplos completos.**

### 6A — Dependency Rule violations

Verificar `.csproj` references e `using` statements:
- ❌ `Domain` referenciando `Infrastructure` ou `Application`
- ❌ `Application` referenciando `Infrastructure` diretamente
- ❌ `[Key]`, `[Required]`, `[Column]` do EF Core em Domain entities
- ❌ `HttpContext` ou `IHttpContextAccessor` em Application/Domain

### 6B — Fat Controllers

Sinalizar Controllers com:
- Mais de 15 linhas por action
- Validação de negócio inline
- Acesso direto a DbContext
- 3+ responsabilidades distintas

### 6C — God Services / Classes

Sinalizar quando:
- Classe tem 5+ dependências injetadas não relacionadas
- Método público tem complexidade ciclomática > 10
- Service com 500+ linhas e múltiplos contextos de negócio

---

## STEP 7 — Análise de Performance

**Leia `references/performance.md` para benchmarks e exemplos completos.**

### 7A — LINQ e EF Core

- [ ] `.ToList()` antes de `.Count` → usar `.CountAsync()`
- [ ] `.ToList()` no meio de query chain sem necessidade
- [ ] `Select` sem projeção — carregando entidade completa para DTO simples
- [ ] N+1 queries — lazy loading sem `.Include()` em loops
- [ ] `.First()` sem `OrderBy` em dados não-ordenados
- [ ] `Any()` vs `Count() > 0` (Any é mais eficiente)

### 7B — Async/Await

- [ ] `.Result` ou `.Wait()` → deadlock em ASP.NET
- [ ] `async void` fora de event handlers
- [ ] `await` dentro de `lock` (não compilável, mas verificar `Monitor`)
- [ ] `Task.Run` desnecessário em código já assíncrono
- [ ] Falta de `ConfigureAwait(false)` em bibliotecas
- [ ] `new Task()` em vez de `Task.Run()`

### 7C — Alocações desnecessárias

- [ ] `new byte[]` em hot loops → sugerir `ArrayPool<byte>`
- [ ] Concatenação com `+` em loops → sugerir `StringBuilder`
- [ ] `string.Format` com muitos args → sugerir interpolação (C# 10+: `DefaultInterpolatedStringHandler`)
- [ ] `ILogger` com string interpolation → sugerir message templates ou `LoggerMessage`

### 7D — Tipos e estruturas de dados

- [ ] `Dictionary` para lookups read-only → sugerir `FrozenDictionary` (.NET 8+)
- [ ] `List<T>` quando tamanho é fixo → sugerir array ou `ImmutableArray`
- [ ] `class` para DTO imutável → sugerir `record`
- [ ] `struct` mutável em contexto readonly → sinalizar defensive copies

### 7E — Oportunidades por versão do .NET

Cruzar com versão detectada no Step 2:

| Versão atual | Melhoria disponível | Performance gain |
|---|---|---|
| .NET 5/6 | `MinBy`/`MaxBy` no LINQ (v6) | Elimina `OrderBy().First()` → O(n) em vez de O(n log n) |
| .NET 5/6 | `Chunk()` no LINQ (v6) | Substitui paginação manual com Skip/Take |
| .NET 5/6/7 | `FrozenDictionary` (v8) | ~50% mais rápido em lookups read-only |
| .NET 5/6/7 | `SearchValues<T>` (v8) | Busca vetorizada em strings/arrays |
| .NET 5/6/7 | Primary constructors (v8) | Menos boilerplate, sem impacto de perf |
| .NET 5-7 | `CountBy`/`AggregateBy` (v9) | Substitui `GroupBy` + `Count` — menos alocações |
| .NET 5-8 | `HybridCache` (v9) | L1+L2 cache com stampede protection built-in |
| .NET 5-9 | `field` keyword (v10) | Backing field sem campo explícito |
| < .NET 8 | `LoggerMessage` source gen | Zero-allocation logging |
| < .NET 8 | `IHostedService` → `BackgroundService` | Mais simples e seguro |

---

## STEP 8 — Análise de Resiliência e Segurança

**Leia `references/resilience.md` para exemplos completos.**

### 8A — Retry policies

- [ ] Retry aplicado em erros 4xx (400, 401, 403, 404) → **BUG crítico**
- [ ] Retry sem jitter (thundering herd problem)
- [ ] Retry sem circuit breaker (retry storm)
- [ ] `HttpClient` sem `AddStandardResilienceHandler()` (.NET 8+) ou Polly

### 8B — CancellationToken

- [ ] CancellationToken ausente em métodos async públicos
- [ ] CancellationToken recebido mas não propagado para EF Core / HttpClient
- [ ] `CancellationToken.None` hardcoded em lógica não-trivial

### 8C — Logging

- [ ] `$"User {id} logged in"` → string interpolation com ILogger
- [ ] Dados sensíveis logados (senhas, tokens, PII)
- [ ] Log level incorreto (Exception como `LogWarning`, info de negócio como `LogDebug`)
- [ ] Sem correlation ID / TraceId estruturado

### 8D — Exceptions

- [ ] Exception como control flow para casos esperados (not found, validação)
- [ ] `catch (Exception ex)` genérico sem re-throw ou handling específico
- [ ] `throw ex` em vez de `throw` (perde stack trace)
- [ ] Exceções de domínio herdando de `Exception` sem distinção de tipo

### 8E — Memory e recursos

- [ ] `new HttpClient()` diretamente → socket exhaustion
- [ ] `DbContext` como Singleton
- [ ] `IDisposable` não implementado onde necessário
- [ ] Event handlers não desregistrados (memory leaks)

---

## STEP 9 — Análise de Testes

### 9A — Cobertura de pontos críticos

Verificar se existem testes para:
- [ ] Cálculos de negócio (pricing, impostos, descontos, totais)
- [ ] Transições de estado de Aggregates
- [ ] Validações de Value Objects
- [ ] Lógica de autorização / permissões
- [ ] Edge cases: valores nulos, listas vazias, limites

### 9B — Qualidade dos testes

- [ ] Testes testando o framework, não o comportamento
- [ ] Mocks excessivos (mock de tudo → teste inútil)
- [ ] Setup duplicado sem fixture/builder
- [ ] Assertions sem mensagem descritiva em caso de falha
- [ ] Testes de integração sem rollback de banco

### 9C — Cobertura de novos comportamentos

- [ ] PR adiciona lógica de negócio sem testes correspondentes → **BLOCKER**
- [ ] PR corrige bug sem teste de regressão → sinalizar

---

## STEP 10 — Montar o Relatório Final

Após completar todos os steps, gerar o relatório no formato abaixo.
**Leia `references/report-template.md` para o template completo e exemplos.**

### Estrutura do relatório

```markdown
# Code Review Report

## Metadata
- **Escopo:** PR #42 / Commit abc123 / Arquivo X
- **Data:** YYYY-MM-DD
- **Stack:** .NET X | C# X | Arquitetura | ORM
- **Severidade geral:** 🔴 Blocker / 🟠 High / 🟡 Medium / 🟢 Low

---

## Sumário executivo
[2-4 linhas: o que foi feito, pontos mais críticos, recomendação geral]

---

## Findings

### [CATEGORIA] — [TÍTULO CURTO]
- **Arquivo:** `src/Domain/Orders/Order.cs`
- **Linha:** 42-58
- **Severidade:** 🔴 Blocker | 🟠 High | 🟡 Medium | 🟢 Low | 💡 Sugestão
- **Tipo:** Design Pattern | DDD | Clean Architecture | Performance | Resiliência | Testes | .NET Upgrade
- **Problema:**
  [Descrição clara do problema]
- **Por que é importante:**
  [Referência: Clean Code cap. X / DDD Evans p. X / Microsoft Docs]
- **Código atual:**
  ```csharp
  // código ruim atual
  ```
- **Código sugerido:**
  ```csharp
  // código correto sugerido
  ```
- **Impacto do fix:** [Performance gain / Maintainability / Testability / Compliance]

---

## Oportunidades de upgrade .NET
[Se versão < latest LTS, listar melhorias automáticas por versão]

## Padrões quebrados
[Lista de inconsistências com o padrão do próprio projeto]

## Pontos positivos
[O que foi bem feito — review construtivo inclui elogios]

## Checklist de aprovação
- [ ] Findings 🔴 Blocker resolvidos
- [ ] Findings 🟠 High endereçados ou justificados
- [ ] Testes para nova lógica de negócio
- [ ] Sem breaking changes sem versioning
```

### Regras de severidade

| Severidade | Critério | Ação |
|---|---|---|
| 🔴 Blocker | Bug, security issue, memory leak, retry em 4xx, regra de negócio sem teste | **Não mergeável** |
| 🟠 High | Violação de arquitetura, anemic model, DI lifetime errado, async/await bug | Resolver antes do merge |
| 🟡 Medium | Oportunidade de pattern, primitive obsession, logging incorreto | Resolver nesta sprint |
| 🟢 Low | Naming, documentação, pequenos code smells | Próxima oportunidade |
| 💡 Sugestão | Upgrade de .NET, refactoring opcional, melhoria de DX | Backlog técnico |

---

## Referências

- `references/patterns.md` — Todos os design patterns com exemplos C# ruim vs bom
- `references/ddd.md` — Value Objects, Aggregates, Rich Domain Model, exemplos completos
- `references/architecture.md` — Clean Architecture layers, dependency rule, CQRS
- `references/dotnet-versions.md` — Features e ganhos de performance por versão .NET 5→10
- `references/performance.md` — LINQ, async, alocações, Span<T>, ArrayPool, benchmarks
- `references/resilience.md` — Retry, circuit breaker, CancellationToken, logging, memory leaks
- `references/report-template.md` — Template completo do relatório com exemplos reais
