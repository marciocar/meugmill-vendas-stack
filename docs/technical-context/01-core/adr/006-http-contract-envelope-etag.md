# ADR 006 — Contrato HTTP: `Result<T>`, `vendas/api/`, `If-Match`/`ETag`

> **Status:** Aceito · **Data:** 2026-10-14 · **Última Atualização:** 2026-10-14
> **Nó KG:** `D_KEEP_ETAG_CONCURRENCY`

## Contexto
Hoje: rotas `/v1/...`, resposta = recurso cru ou `{items, nextCursor[, total]}`, erro = `{error, message?, detail?}`,
concorrência otimista por `ETag`/`If-Match` (sem header → 428; versão velha → 409 `version_conflict`).
No Vendas: rotas `vendas/api/...` e envelope `TApiResponseModel<T>` no front / `Result<T>` no back.

## Decisão
- Prefixo **`vendas/api/`** e envelope **`Result<T>`** (via `HttpResultActionFilter` do boilerplate).
- **Mantém** `ETag` na resposta e `If-Match` obrigatório nas escritas versionadas; mantém 428 e 409 `version_conflict`.
- O **código de erro de domínio** (`portfolio_incomplete`, `link_conflict`…) continua existindo — o front traduz por
  código (`apps/web/src/api/errors.ts`). Ele vai em `errors[]` e o `detail` (contagens, ids de carteira) num campo do envelope.
- Paginação por cursor continua: `value = { items, nextCursor }`, `count = total` quando houver.

## Divergência registrada (resolver na implementação)
| Campo | Boilerplate back (`Core/Helpers/Result.cs`) | Boilerplate front (`TApiResponseModel`) |
|---|---|---|
| data da requisição | `DataRequisicao` (`DateTime.Now`, hora local) | `requestDate?` |
| erros | `IList<string>` | `unknown[]` |
| status | `HttpStatusCode` (enum → string com `JsonStringEnumConverter`) | `httpStatusCode: string` |

Recomendação: seguir o que a API Vendas **realmente** serializa hoje; preferir `DateTime.UtcNow`.

## Consequências
- ✅ Axios/hooks do Vendas consomem sem adaptação de formato.
- ⚠️ O Axios do Vendas precisa **expor** `ETag` (CORS `exposeHeaders`) e enviar `If-Match` por chamada.
- ⚠️ `Result.WithError` mapeia tudo para 400 e o filtro só converte 400/404 — 403, 409 e 428 precisam de caminho próprio
  (estender `Result` ou o filtro). Sem isso, os códigos do legado viram 400.
- ⚠️ A **ordem dos erros** é contrato: 404 (não lê) → 403 (não edita) → 403 (troca de filial/responsável) → 428 → 409 `portfolio_inactive` → 409 `version_conflict`.
- ⚠️ `Content-Disposition` precisa estar exposto para a exportação CSV.
