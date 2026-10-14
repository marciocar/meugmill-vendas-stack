# Codebase Guide — AS-IS, TO-BE e mapa de migração

> **Última Atualização:** 2026-10-14 · fonte: este repo (commit `0b0b2cf`) e `~/boilerplates/gmill/meugmill/{back,front}`

## AS-IS — este repositório

```
apps/api/                 API Node 22 + Fastify 5
├─ src/server.ts          entrypoint: loadConfig → buildApp → listen; SIGTERM/SIGINT com timeout 10 s
├─ src/app.ts             ordem dos plugins: observability → cors → db → auth → openapi → health → me → v1
├─ src/config.ts          env vars (schema TypeBox) — ver contributing
├─ src/plugins/           observability (pino, redact LGPD, request-id, erros) · db (SQLite, pragmas, migrate no boot)
│                         · auth (jose, JWKS, fail-closed) · openapi (@fastify/swagger)
├─ src/routes/            health, me, v1/* (1 arquivo por recurso; crud.ts gera CRUD; http.ts = If-Match/ETag/erros)
├─ src/domain/            regras (síncronas, better-sqlite3)
│  ├─ shared/             errors, authz (Actor, escopo), pagination, db (writeTx immediate), links, normalize, cnpj, audit
│  ├─ portfolios/         agregado, acesso, authz, name-key, validate
│  ├─ eligibility/        query.ts = motor SQL de prévia e disputa
│  ├─ conflicts/          membros efetivos
│  ├─ distribution/       grid, service, strategies/balanced
│  ├─ links/              finalize, write (vínculos + outbox)
│  ├─ visibility/         profiles, sql, service
│  ├─ csv/                codec, layouts, jobs, export, import/*
│  └─ branches, catalog, customers, sellers, portfolio-types, geo   (cadastros)
├─ src/db/schema.ts       schema Drizzle (~23 tabelas)
├─ drizzle/               13 migrations (0002 = seed IBGE; 0006 vazia)
├─ test/                  Vitest: raiz (infra), routes/, domain/ (+ helpers jwt.ts, seed.ts)
└─ openapi-v1.json        contrato versionado
apps/web/                 custom element <gmill-carteira> (React 19, Vite library mode → dist/gmill-carteira.js)
├─ src/element.tsx        registro, Shadow DOM, props apiBase/token, evento token-expired
├─ src/App.tsx, Shell.tsx /v1/me, abas Carteiras · Meus clientes · Importar e exportar
├─ src/portfolios/        Wizard + Steps
├─ src/csv/, visibility/  CSV e "meus clientes"
├─ src/api/               client.ts (fetch + If-Match + ApiError), errors.ts (códigos → pt-BR), types.ts
├─ demo/                  host de demonstração (token do IdP mock)
└─ public/docs/           manual do usuário (manual.md), api.html (Scalar), público via nginx
packages/shared/          tipos UserClaims/Actor (o web NÃO importa)
docker/                   api/web Dockerfiles, nginx.conf, idp-config.json (mock-oauth2-server)
compose.yaml              idp + api + web, portas em 127.0.0.1 (39000/39080/39081)
compose.gmill.yaml        override da demo pública gmill.onionevolve.com (atrás de Caddy com basic auth)
scripts/smoke.sh          E2E por curl (E2–E10) · scripts/demo/seed.mjs carga fictícia idempotente
```

**Fluxo de uma requisição:** `onRequest: authenticate` (401 antes de validar corpo) → validação TypeBox → handler →
`actorOf(request)` → service de domínio (autoriza dentro) → `writeTx` → `sendDomainError` mapeia `DomainError` → status.

## TO-BE — back (boilerplate GMill, .NET 10)

```
Api.sln · Directory.Build.props (SonarAnalyzer.CSharp)
src/Core/      entidades, Commands/Queries + Handler/, Behaviors/ValidationBehavior, Models/{Requests,Responses,Filters,Validations},
               Mapping/ (IRegister), Interfaces/{Repositories,...}, Helpers/Result.cs, Security/, Enums/
src/Infra/     Data/AppDbContext, ModelConfiguration/, Repositories/, Migrations/, UnitOfWork, DesignTimeDbContextFactory
src/Web/       Program.cs (minimal hosting + Autofac), Controllers/ (BaseApiController : ControllerBase),
               Extension/ (Jwt, Swagger, Serilog, Redis, Policy, Mapping, EntityConfiguration...), Filters/HttpResultActionFilter,
               Handlers/GlobalExceptionHandler
tests/Core.Tests/  xUnit + Moq: Commands/, Queries/, Behaviors/, Mapping/, TestHelpers/MapperTestFactory
```

## TO-BE — front (feature no Vendas Front)
```
src/data/services/portfolios/<acao>.ts + index.ts     chamadas via api (Axios único)
src/data/hooks/use-<nome>.ts                         useQuery/useInfiniteQuery/useMutation
src/data/schema/portfolios/index.ts                  Zod (substitui src/api/types.ts duplicado)
src/data/types/portfolio.ts                          T*/I* com TApiResponseModel<T>
src/shared/enums/key.ts                              EnumKeyPortfolio { get_all = 'portfolio_get_all', ... }
src/pages/authenticated/carteiras/...                ver ADR 002
```

## Mapa "de → para"

| AS-IS | TO-BE | Nota |
|---|---|---|
| `routes/v1/<recurso>.ts` | `Web/Controllers/<Recurso>Controller.cs` | rota `vendas/api/...`; `[Authorize]` |
| `domain/<x>/service.ts` (métodos) | `Core/Commands|Queries/<X>/…` + `Handler/` | handlers retornam `ValueTask<T>` |
| TypeBox schemas | `Models/Requests` + `Models/Validations` (FluentValidation) | validação no `ValidationBehavior` |
| `withRoleGuard` | `IPipelineBehavior` de perfil | 403 `role_case_mismatch` |
| `DomainError` + `sendDomainError` | `Result<T>` + filtro estendido / `GlobalExceptionHandler` | cobrir 403/409/428 (ADR 006) |
| `writeTx` immediate | transação EF + lock por filial | ADR 005 |
| `eligibility/query.ts`, `visibility/sql.ts` | SQL cru Postgres em `Infra/Data/Queries` (Dapper ou `SqlQuery`) | não é LINQ |
| `db/schema.ts` + `drizzle/` | `Infra/Data/ModelConfiguration` + `Migrations` | tipos explícitos (`HasColumnType`) |
| `csv/jobs.ts` (fila em memória) | `BackgroundService` + `import_jobs` no Postgres | ADR 007 |
| `plugins/auth.ts` (jose) | `JwtBearer` com validação completa | ADR 003/010 |
| `plugins/observability.ts` (pino redact) | Serilog + destructuring/masking | manter redact de token/CNPJ/SQL |
| `config.ts` env | `SettingsConfigurationHelper` ("env vence appsettings") | segredos fora do repo |
| cadastros CRUD | réplica + sincronizador | ADR 004 |
| `apps/web/src/api/client.ts` | `data/libs/axios.ts` (já existe no Vendas) + header `If-Match` por chamada | |
| `useAsync` / `usePaged` / `listAll` | `useQuery` / `useInfiniteQuery` | |
| polling `ImportJobPanel` | `refetchInterval` com retry só em transitório | |
| CSS `gc-*`, `Notice`, `Field`, `Dropdown` | AntD (Alert, Form, Table, Steps, Dropdown, Modal.confirm, Upload) + Tailwind | |
| abas montadas + dirty-guard | rotas + `useBlocker` | |
| `errors.ts` (código → pt-BR) | helper de erro + `useNotificationStore` | mensagens fixas, sem eco |
