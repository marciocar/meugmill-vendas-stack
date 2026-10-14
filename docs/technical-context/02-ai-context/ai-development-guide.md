# AI Development Guide — Carteira de Clientes

> **Última Atualização:** 2026-10-14 · para quem (humano ou IA) for implementar o port. Leia junto com
> [business-logic](../03-domain/business-logic.md) e os ADRs.

## Regras de ouro
1. **O legado é a especificação.** Antes de portar uma regra, leia o service e o teste de domínio correspondentes
   (`apps/api/src/domain/<x>/`, `apps/api/test/domain/<x>*.test.ts`). Não "simplifique" uma regra sem ADR.
2. **Siga o boilerplate, menos onde o ADR 010 corrige.** JWT sempre com validação completa; nenhum segredo em `appsettings`.
3. **Código em inglês, textos em pt-BR** (`CLAUDE.md`). Nomes do boilerplate em pt-BR (`ParametroSistema`, `RetornaValor`) não são modelo de nomenclatura.
4. **Licenças:** Mapster (não AutoMapper), Mediator martinothamar (não MediatR), xUnit `Assert` (não FluentAssertions 8+).

## Padrões do back alvo
- **Command/Query + Handler** por caso de uso; handler retorna `ValueTask<Result<T>>`.
- **Validação** com `AbstractValidator<TRequest>` registrado por assembly; roda no `ValidationBehavior`. `ValidationException` → 400 no `GlobalExceptionHandler`.
- **Autorização de domínio dentro do handler** (como no legado), não só em atributo: escopo de filial, perfil, responsável.
- **Mapeamento** em classes `IRegister`; nos testes, mapper real (`MapperTestFactory`), não mock.
- **Async de ponta a ponta** (`MigrateAsync`, `RunAsync`, EF async).
- **SQL cru** para o motor (elegibilidade, disputa, visibilidade): parâmetros sempre (`@p`), arrays via `= ANY(@ids)`; nunca concatenar.

## Gotchas do port (os que mais quebram)
| Tema | Armadilha |
|---|---|
| Booleano | `case when code & 1` não vale no Postgres: `(code & 1) <> 0` |
| Contagem | `sum(kind = 'include')` → `count(*) filter (where kind = 'include')` |
| Colação | desempate do `balanced` e `order by code` são **ordinais**: `COLLATE "C"` e `StringComparer.Ordinal` |
| Maiúsculas | `ToUpperInvariant` ≠ `toUpperCase` do JS (`ß`); replicar `searchKey` com teste de tabela |
| Datas | epoch ms não cabe em `int4`; `DateTime.Now` do `Result` é hora local — preferir UTC |
| Serialização | o legado assume 1 escritor; no Postgres use lock por filial em finalize, distribute, troca de filial e CSV |
| Eventos | nunca `id > max(id)` para achar o que acabou de criar: `RETURNING` |
| Outbox | sequence ≠ ordem de commit (ADR 008) |
| Erros | `Result.WithError` = 400 sempre; 403/409/428 precisam de caminho próprio. `WithException` (500) **não** é convertido pelo filtro e sai 200 |
| No-op | operação sem mudança não sobe versão e não exige `If-Match` em alguns casos (`applyLinkActiveTransition`) |
| Ordem dos erros | 404 → 403 → 403 troca → 428 → 409 inativa → 409 versão (testes do legado afirmam) |

## Segurança e LGPD
- Claims mínimas; nunca logar token, CNPJ, e-mail, query string, nem a mensagem de exceção do EF/Npgsql (pode conter SQL e parâmetros).
- Fora do escopo = 404; mensagens fixas sem eco de dado (`"Cliente inválido ou fora do escopo"`).
- CSV: proteção contra fórmula na exportação; conteúdo do arquivo nunca no banco de domínio.
- `VISIBILITY_LEGACY=deny` em produção.
- CORS só com origens exatas; nada de `*` (o SonarAnalyzer `S5122` acusa).

## Desempenho
Use os tetos de [business-logic §Desempenho](../03-domain/business-logic.md#desempenho) como meta e meça no Postgres.
Índices compostos do legado estão comentados em `apps/api/src/db/schema.ts` — portá-los junto com o schema.

## Front alvo (Vendas Front)
- HTTP só via `api` de `@/data/libs/axios`; `If-Match` por chamada; tratar `version_conflict` relendo a carteira.
- Query keys por feature em `shared/enums/key.ts`; invalidar no `onSuccess`.
- Formulários RHF + Zod; tipos derivados do schema.
- Permissões de tela (admin, admin da filial, responsável) espelham `apps/web/src/roles.ts`, mas **a API é a autoridade**.
