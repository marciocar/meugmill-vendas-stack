# Troubleshooting — Carteira de Clientes

> **Última Atualização:** 2026-10-14 · problemas reais visíveis no código/config AS-IS e armadilhas já identificadas no boilerplate TO-BE.

## AS-IS (stack atual)

| Sintoma | Causa | Ação |
|---|---|---|
| API não sobe: `ConfigError` | `OIDC_ISSUER`/`OIDC_AUDIENCE` ausentes, ou issuer/JWKS em `http` sem `OIDC_ALLOW_INSECURE_HTTP=true` | definir as variáveis (`apps/api/src/config.ts`) |
| Toda chamada → **503 `auth_unavailable`** | discovery/JWKS do IdP inacessível; cooldown de 10 s repete a falha | conferir o IdP e `OIDC_JWKS_URI`; aguardar o cooldown |
| 401 com token do IdP local | `iss` do mock é montado do Host da requisição (`localhost:39080`); com o override `compose.gmill.yaml` ativo o issuer muda | subir sem o override para uso local |
| Mudança em `docker/idp-config.json` sem efeito | container do mock não recriado | `docker compose up -d --force-recreate idp` |
| 403 para usuário que deveria ser admin | papel com caixa/espaço diferente (`"Admin"`) | ver log `role_case_mismatch`; corrigir o papel no IdP |
| Usuário sem perfil lê a filial inteira | `VISIBILITY_LEGACY=allow` (padrão) | `deny` fora de dev (decisão `D_VISIBILITY_LEGACY_DENY_PROD`) |
| Registro "some" (404) para um usuário | fora do escopo de filial → 404 de propósito | conferir `branch_ids` do token |
| 428 `precondition_required` | escrita versionada sem `If-Match` | mandar a versão do último `ETag` |
| Upload de CSV recusado antes da API | limite do nginx (padrão 1 MB; configurado 17 MB) | `client_max_body_size` no proxy |
| `too_many_imports` | 5 jobs abertos/48 MB por usuário ou 256 MB total | cancelar/confirmar jobs abertos |
| Jobs `interrupted` após restart | fila e arquivo ficam em memória | reenviar o arquivo |
| CI falha em `openapi.test.ts` | `openapi-v1.json` desatualizado | `pnpm --filter @meugmill/api openapi:export` |
| Erro de escrita `SQLITE_BUSY` com várias réplicas | SQLite tem 1 escritor | não escalar a API além de 1 instância |
| Deep link no web dá 404 | nginx não faz fallback de SPA (é bundle de componente) | não se aplica ao alvo |

## TO-BE (armadilhas do boilerplate e do port)

| Sintoma | Causa | Ação |
|---|---|---|
| Token vencido/forjado aceito | `Validate*=false` no `JwtSecurityExtension` do boilerplate | ADR 010: validação completa |
| Erro 500 de domínio chega como **HTTP 200** | controllers devolvem `Ok(response)` e o `HttpResultActionFilter` só converte 400/404 | estender o filtro para 403/409/428/500 (ADR 006) |
| Rotas em `api/...` e não `vendas/api/...` | `BaseApiController` usa `[Route("api/[controller]")]` | confirmar se o prefixo `vendas/` vem do gateway/`PathBase` ou do atributo `[TO BE COMPLETED]` |
| 401 em toda rota exceto `/api/login` | `TokenManagerMiddleware` exige token "ativo" no Redis | não registrar esse middleware neste serviço (ADR 003) |
| `ValidationException` vira 500 | `GlobalExceptionHandler` sem o caso de FluentValidation | tratar e devolver 400 (`BOILERPLATE.md` §4) |
| Migration inesperada ao subir Npgsql | convenção do provider mudou (ex.: `timestamptz`) | `has-pending-model-changes`; fixar `HasColumnType` (`BOILERPLATE.md` §6) |
| Duplicidade/ordem estranha em vínculos sob carga | falta de lock por filial; eventos achados por `max(id)` | ADR 005/008 |
| Desempate de vendedores diferente do legado | comparação com cultura em vez de ordinal | `StringComparer.Ordinal`, `COLLATE "C"` |
| `searchKey` diferente do legado | `ToUpperInvariant` × `toUpperCase` do JS | replicar a função e testar com tabela de casos |
| `ETag` invisível no front | CORS sem `exposeHeaders` | expor `ETag`, `Content-Disposition`, `X-Request-Id` |
