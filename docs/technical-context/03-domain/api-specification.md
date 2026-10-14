# API Specification — Carteira de Clientes

> **Última Atualização:** 2026-10-14 · fonte AS-IS: `apps/api/openapi-v1.json` (84 operações, 65 paths), `apps/api/src/routes/`.
> TO-BE: ADR [003](../01-core/adr/003-auth-vendas-token.md), [004](../01-core/adr/004-master-data-local-replica.md),
> [006](../01-core/adr/006-http-contract-envelope-etag.md), [007](../01-core/adr/007-csv-portfolios-links-background-job.md).

O contrato versionado é `apps/api/openapi-v1.json`; o teste `test/routes/openapi.test.ts` falha se ele divergir do gerado
(`pnpm --filter @meugmill/api openapi:export`). Documentação navegável (AS-IS): `/docs/api.html` (Scalar).

## Convenções AS-IS

| Tema | Regra |
|---|---|
| Prefixo | `/v1/...`; `/health` e `/ready` sem prefixo |
| Auth | `Authorization: Bearer <JWT>` em tudo, exceto `/health`, `/ready`, `/v1/openapi.json` |
| Sucesso | recurso cru; listas `{ items, nextCursor[, total] }` |
| Erro | `{ error: "<code>", message?, detail? }` — `message` só em `validation_error`; `detail` só contagens/ids, nunca dado de cliente |
| Paginação | `q, active, cursor, limit` (padrão 50, máx. 200); cursor = base64url do último id |
| Concorrência | `ETag: "<version>"` em get/create/patch/transições; `If-Match` aceita `"3"`, `3`, `W/"3"` |
| Request id | `X-Request-Id` aceito se `^[A-Za-z0-9._-]{1,128}$`, senão UUID gerado; ecoado sempre |
| CORS | origens exatas (`CORS_ORIGINS`, sem `*`); expõe `x-request-id`, `ETag`, `Content-Disposition` |

## Códigos de erro (`domain/shared/errors.ts`)

| Status | Códigos |
|---|---|
| 400 | `validation_error`, `bad_request` |
| 401 | `unauthorized` (+ `WWW-Authenticate: Bearer`) |
| 403 | `forbidden` |
| 404 | `not_found` (inclusive "fora do escopo") |
| 409 | `conflict`, `customer_exists`, `seller_exists`, `portfolio_inactive`, `portfolio_incomplete`, `portfolio_has_conflicts`, `link_conflict`, `version_conflict`, `import_not_ready`, `too_many_imports` |
| 428 | `precondition_required` |
| 500 | `internal_error` |
| 503 | `auth_unavailable` (não foi possível validar o token), `unavailable` (`/ready`) |

## Endpoints AS-IS

Legenda da coluna TO-BE: ✅ fica · 🔁 vira leitura da réplica · ❌ sai (cadastro é do Vendas/ERP).

### Infra e identidade
| Método | Path | Auth | Resposta | TO-BE |
|---|---|---|---|---|
| GET | `/health` | — | `{status:'ok'}` | ✅ (`/status` do boilerplate) |
| GET | `/ready` | — | `{status:'ready'}` / 503 | ✅ health check do Postgres |
| GET | `/v1/openapi.json` | — | OpenAPI 3.0.3 | ✅ Swagger/ReDoc |
| GET | `/v1/me` | bearer | `{sub, roles[], branchIds[]}` | ✅ |

### Cadastros (CRUD gerado por `routes/v1/crud.ts`)
Recursos: `/v1/branches`, `/v1/product-subgroups`, `/v1/retail-networks`, `/v1/economic-groups`, `/v1/portfolio-types`, `/v1/sellers`, `/v1/customers`.
Operações: `GET {p}` · `GET {p}/:id` · `POST {p}` (201) · `PATCH {p}/:id` · `POST {p}/:id/deactivate|reactivate`; sellers/customers também
`/deactivate-global|reactivate-global`, `POST /v1/customers/by-cnpj/{cnpj}/branches`, `POST /v1/sellers/by-code/{code}/branches`.
Escrita só `admin`. **TO-BE:** leituras 🔁 (listas e get para o wizard: filiais, subgrupos, redes, grupos, tipos, vendedores); escritas ❌.

### Geo
| Método | Path | Auth | Resposta | TO-BE |
|---|---|---|---|---|
| GET | `/v1/geo/states` · `/v1/geo/municipalities?uf&q` | bearer | lista / paginada | ✅ |

### Carteiras
| Método | Path | Permissão | Resposta | TO-BE |
|---|---|---|---|---|
| GET | `/v1/portfolios?q&branchId&status&active&responsibleSub` | visível | paginada | ✅ |
| POST | `/v1/portfolios` | admin com a filial | agregado (201) | ✅ |
| GET | `/v1/portfolios/:id[?include=conflicts]` | visível | agregado `{branch, type, sellers, filters, overrides…, conflictsBlocked/Lost, status, finalizedAt}` | ✅ |
| PATCH | `/v1/portfolios/:id` | admin ou responsável; filial/responsável só admin | agregado | ✅ |
| PUT | `/v1/portfolios/:id/filters` · `/sellers` | edição + `If-Match` | agregado | ✅ |
| POST | `/v1/portfolios/:id/deactivate` · `/reactivate` | admin | agregado | ✅ |
| GET | `/v1/portfolios/:id/preview?q&source&resolution` | leitura interna | `{items, nextCursor, total}` | ✅ |
| GET/PUT | `/v1/portfolios/:id/overrides` | PUT: edição + `If-Match` | `{include, exclude}` | ✅ |
| GET/PUT | `/v1/portfolios/:id/assignments?productSubgroupId&sellerId&status` | PUT: edição + `If-Match` | paginada + `total` | ✅ |
| GET | `/v1/portfolios/:id/assignments/summary` | leitura interna | `{subgroups, totals}` | ✅ |
| POST | `/v1/portfolios/:id/distribute` | edição + `If-Match` | `{distributed, finalCounts, portfolio, skippedSubgroupIds}` | ✅ |
| POST | `/v1/portfolios/:id/finalize` | edição + `If-Match` | `{created, ended, kept, takenOver, portfolio}` | ✅ |
| GET | `/v1/portfolios/:id/links` · `/links/history` | leitura interna | paginada + `total` | ✅ |

### Vínculos, visibilidade
| Método | Path | Permissão | Resposta | TO-BE |
|---|---|---|---|---|
| GET | `/v1/link-events?branchId&after&limit` | leitura ampla | `{items, nextAfter, hasMore}` | ✅ entrega segura (ADR 008) |
| GET | `/v1/me/visibility` | bearer | `{mode, profiles, byProfile, seller, visibleCustomers}` | ✅ |
| GET | `/v1/me/customers?q&productSubgroupId` | bearer | `{items, nextCursor, total}` | ✅ |
| POST | `/v1/visibility/check` | bearer | `{visible}` (≤ 1.000 ids) | ✅ |

### CSV
| Método | Path | Permissão | Resposta | TO-BE |
|---|---|---|---|---|
| GET | `/v1/csv-layouts` · `/v1/csv-layouts/:layout` | bearer | `{format, layouts[]}` | ✅ só carteiras/vínculos |
| POST | `/v1/imports?layout=` (corpo `text/csv` ≤ 16 MB) | admin, checado **antes** de ler o corpo | 202, job `validating` | ✅ só carteiras/vínculos |
| GET | `/v1/imports` · `/:id` · `/:id/lines` | criador | job / linhas | ✅ |
| POST | `/v1/imports/:id/confirm` (202) · `/cancel` | admin | job | ✅ |
| GET | `/v1/exports/:layout` | escopo + visibilidade | `text/csv; charset=utf-8` com BOM, `Content-Disposition`, `no-store` | ✅ (cadastros: avaliar) |

## Contrato TO-BE (ADR 006)

- **Prefixo:** `vendas/api/<recurso>` — mapa sugerido: `/v1/portfolios` → `vendas/api/Portfolio`, `/v1/me/customers` → `vendas/api/Portfolio/my-customers`
  `[TO BE COMPLETED]` (seguir a convenção de nomes de controller da API Vendas).
- **Envelope `Result<T>`:**
  ```json
  { "value": { "items": [], "nextCursor": null }, "count": 0,
    "hasSuccess": true, "hasError": false, "errors": [], "httpStatusCode": "OK" }
  ```
  Erro de domínio: `hasError: true`, `errors: ["portfolio_incomplete"]`, `detail` em campo adicional, status HTTP real (409, 428…).
- **Mantidos:** `ETag`/`If-Match`, 428, 409 `version_conflict`, ordem dos erros, 404 para fora do escopo, `X-Request-Id`.
- **Auth:** JWT do Vendas (ADR 003); 401 sem token válido; 503 se a chave/JWKS não puder ser obtida (fail-closed).
- **CORS:** origens do Vendas Front por ambiente `[TO BE COMPLETED]`; expor `ETag`, `Content-Disposition`, `X-Request-Id`; permitir `If-Match`.
