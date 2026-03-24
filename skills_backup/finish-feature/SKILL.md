---
name: finish-feature
description: >-
  Finalize a .NET C# feature: run full Release build and test suite, update all
  documentation (SDD, spec, plan, OpenAPI/Scalar annotations, README, ADRs),
  clean up temporary files and debug code, and create a proper git commit
  without any AI co-author attribution. Use this skill whenever the user says
  "finish feature", "finalizar", "wrap up", "fechar feature", "preparar PR",
  "terminar", "limpar e commitar", or "done". This skill is the mandatory
  quality gate before a feature is considered complete — never skip it.
---

# Finish Feature

## Overview
Final quality gate antes de uma feature ir para review. Cada passo é
obrigatório — um passo pulado é uma dívida deixada para o próximo dev.

---

## Step 1: BUILD RELEASE

```bash
dotnet build --configuration Release
```
- Zero erros, zero warnings
- Trate warnings como erros — corrija todos antes de prosseguir

---

## Step 2: SUITE COMPLETA DE TESTES

```bash
dotnet test --configuration Release --verbosity normal
```
- 100% dos testes passando (unitários + integração)
- Registre: total / passando / falhando / pulados
- Se há `[Skip]`: avalie se deve ser reabilitado e informe o usuário

---

## Step 3: VERIFICAÇÃO DE FORMATAÇÃO

```bash
dotnet format --verify-no-changes
dotnet format analyzers --verify-no-changes 2>/dev/null || true
```
Se houver problemas: `dotnet format` e inclua no commit.

---

## Step 4: COBERTURA DE TESTES (opcional mas recomendado)

```bash
dotnet test --collect:"XPlat Code Coverage" --results-directory ./coverage
# Gerar relatório legível (requer reportgenerator)
dotnet tool run reportgenerator \
  -reports:"./coverage/**/coverage.cobertura.xml" \
  -targetdir:"./coverage/report" \
  -reporttypes:TextSummary 2>/dev/null && cat ./coverage/report/Summary.txt || true
```

Verifique cobertura da feature nova:
- Busque os arquivos criados nesta feature e avalie se há paths não cobertos
- Cobertura < 70% nos handlers → adicione testes faltantes antes de continuar
- Não force 100% de cobertura — avalie qualidade dos testes, não só quantidade

---

## Step 5: SCAN DE SECRETS E CREDENCIAIS

```bash
# Verificar segredos expostos no diff
git diff HEAD -- "*.cs" "*.json" "*.yaml" "*.yml" | \
  grep -iE "(password|secret|token|api.?key|connectionstring)\s*=\s*['\"][^'\"]" | \
  grep -v "appsettings.Development" | \
  grep -v "placeholder\|example\|your_" || echo "Nenhum secret detectado ✅"

# Verificar connection strings hardcoded em código C#
grep -rn "Server=\|Data Source=\|mongodb://\|redis://" src/ --include="*.cs" 2>/dev/null | \
  grep -v "//.*Server=\|test\|Test" || echo "Sem connection strings hardcoded ✅"

# Verificar API keys
grep -rn "sk-\|Bearer [a-zA-Z0-9]\{40\}" src/ --include="*.cs" 2>/dev/null | \
  head -5 || echo "Sem tokens hardcoded ✅"
```

Se encontrar secrets: **PARE imediatamente**. Não faça commit. Remova o secret, use `IConfiguration` e informe o usuário.

---

## Step 6: LIMPEZA DE CÓDIGO

```bash
# Código de debug esquecido
grep -rn "Console\.Write\|Debug\.Print\|Debugger\.Break\|\.Dump()" \
  src/ --include="*.cs" 2>/dev/null

# TODOs e HACKs
grep -rn "// TODO\|// HACK\|// FIXME" src/ --include="*.cs" 2>/dev/null

# Limpar arquivos temporários
find . -name "*.tmp" -o -name "debug-*.txt" -o -name "*.bak" 2>/dev/null | \
  xargs rm -f 2>/dev/null && echo "Arquivos temporários removidos ✅"
```

Checklist:
- [ ] Nenhum `Console.WriteLine` / `Debug.Print` / `Debugger.Break()`
- [ ] Nenhum bloco de código comentado (Git guarda a história)
- [ ] Nenhum `// TODO` sem issue associada
- [ ] Nenhuma string hardcoded que deveria estar em `IConfiguration`

---

## Step 7: ATUALIZAÇÃO DE DOCUMENTAÇÃO

### 7.1 Fechar artefatos da feature
`docs/design/FEAT-{name}/spec.md`:
```markdown
## Status: ✅ Implementado ({YYYY-MM-DD})
- [x] CA-01: {descrição} ✅
```

`docs/design/FEAT-{name}/plan.md`:
- Confirme `[x] ✅` em todas as tarefas
- Adicione: `**Concluído em:** {YYYY-MM-DD}`

### 7.2 OpenAPI/Scalar — anotações completas
```csharp
group.MapPost("/", handler)
    .WithName("Create{Entity}")
    .WithTags("{Entities}")
    .WithSummary("Cria um novo {entity}")
    .WithDescription("Descrição detalhada do comportamento.")
    .Produces<{ResponseDto}>(StatusCodes.Status201Created)
    .ProducesValidationProblem()
    .Produces<ProblemDetails>(StatusCodes.Status400BadRequest)
    .Produces(StatusCodes.Status401Unauthorized)
    .RequireAuthorization();
```
Verifique: `curl -s http://localhost:5000/scalar/v1 -o /dev/null -w "%{http_code}"`

### 7.3 CHANGELOG.md
Adicione entrada seguindo o formato Keep a Changelog:
```markdown
## [Unreleased]

### Added
- {Nome da Feature}: {descrição breve do que foi adicionado}

### Changed (se aplicável)
- {O que mudou em comportamento existente}
```
Se não existir `CHANGELOG.md`, crie-o com a estrutura inicial.

### 7.4 README.md (se necessário)
Atualize se a feature introduziu:
- Novas variáveis de ambiente (`{NOVA_VAR}` — descrição)
- Novos passos de setup
- Novos pré-requisitos

### 7.5 ADR (se decisão arquitetural foi tomada)
Crie `docs/adr/{NNNN}-{titulo}.md` se houver decisão técnica significativa:
```markdown
# {NNNN}. {Título}
**Status:** Aceito ({YYYY-MM-DD})
## Contexto: {por que a decisão foi necessária}
## Decisão: {o que foi escolhido e por quê}
## Consequências: ✅ {positiva} / ⚠️ {tradeoff}
```

---

## Step 8: REVISÃO DO DIFF

```bash
git diff --stat HEAD
git status
# Revisar mudanças em arquivos .cs
git diff HEAD -- "*.cs" | head -300
```

Confirme:
- [ ] Nenhum secret ou credencial exposta
- [ ] Nenhuma mudança não relacionada à feature
- [ ] Todos os arquivos novos rastreados pelo git

---

## Step 9: COMMIT FINAL

```bash
git add -A
git commit -m "feat({scope}): {descrição concisa no imperativo}

{Opcional: 1-2 frases sobre o que foi implementado}

Closes #{numero-do-issue}"
```

**Regras absolutas:**
- ❌ NÃO adicione `Co-Authored-By`
- ❌ NÃO adicione `Generated with Claude` ou similar
- ✅ Subject line: minúsculas, imperativo, máx 72 chars
- ✅ Referencie o issue/ticket

Se há commits de fixup para consolidar:
```bash
git rebase -i HEAD~{N}
# squash os fixups, mantenha commits significativos
```

---

## Step 10: SUMÁRIO FINAL

```
## Feature Concluída: {Nome} ✅

### Verificações
- ✅ Build Release: 0 erros, 0 warnings
- ✅ Testes: {X}/{X} passando
- ✅ Formatação: limpa
- ✅ Secrets: nenhum exposto
- ✅ Código: sem debug, sem TODOs pendentes
- ✅ Docs: SDD/spec/plan fechados, CHANGELOG atualizado
- ✅ OpenAPI/Scalar: anotações completas
- ✅ Git: commit convencional, sem atribuição de IA

### Resumo das Mudanças
- Arquivos criados: {N} — {lista principal}
- Arquivos modificados: {N} — {lista principal}
- Testes adicionados: {N} (unitários: X, integração: Y)
- Migrations: {lista ou "nenhuma"}
- Documentação: {lista do que foi atualizado}

### Próximos Passos
- {Follow-ups ou limitações conhecidas}
- Branch pronta para PR
```

---

## Guardrails
- NÃO pule nenhum step
- NÃO faça commit se secrets forem encontrados — remova primeiro
- NÃO faça commit com Co-Authored-By ou atribuição de IA
- Se build ou testes falharem: corrija — sem exceções
- Se mudanças significativas forem necessárias durante limpeza: informe o usuário
