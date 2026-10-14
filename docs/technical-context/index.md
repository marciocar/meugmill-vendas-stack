# Contexto Técnico — MeuGmill Vendas (Carteira de Clientes)

> **Última Atualização:** 2026-10-14 · gerado por `/docs:build-tech-docs` · decisões seladas por `/meta:decide`
> no grafo [`docs/onion/graph/onion-adoption.kg.yaml`](../onion/graph/onion-adoption.kg.yaml) (nós `D_*`).

Este é o **único ponto de entrada** do contexto técnico. O repositório contém hoje o **legado** (Node/TS) que
será **transformado** para o padrão de stack da GMill. A documentação descreve as duas pontas:

- **AS-IS (legado):** o que executa hoje — vale como **especificação de comportamento** do domínio.
- **TO-BE (alvo):** microsserviço .NET 10 (boilerplate GMill back) + feature no Vendas Front (boilerplate GMill front).
- **Mapa de migração:** o que muda, o que se preserva e o que é risco.

## Perfil técnico

| Item | AS-IS (este repo) | TO-BE (alvo) |
|---|---|---|
| Tipo | Microsserviço + componente web embarcável | Microsserviço + feature de SPA |
| Back | Node 22 · TypeScript ~5.9 · Fastify 5 · TypeBox · Drizzle · better-sqlite3 · jose | .NET 10 · ASP.NET Core · Clean Architecture (Core/Infra/Web) · Autofac · Mediator · Mapster · FluentValidation · EF Core + Npgsql · Serilog |
| Banco | SQLite (arquivo único, WAL, 1 escritor) | Postgres próprio (`D_DB_OWN_POSTGRES`) |
| Front | Custom element `<gmill-carteira>` (React 19 puro, Shadow DOM) | Feature do Vendas Front: React 19 · AntD 5 · Tailwind 4 · TanStack Query · Zustand · RHF+Zod · Router 7 · Axios · MSAL · Pusher (`D_FRONT_IN_VENDAS_FRONT`) |
| Auth | OIDC genérico (JWKS, assimétrico, fail-closed), claims `roles` + `branch_ids` | JWT emitido pela API Vendas (`D_AUTH_VENDAS_TOKEN`) |
| Contrato | `/v1/...`, recurso cru, erro `{error,message?,detail?}`, `If-Match`/`ETag` | `vendas/api/...`, envelope `Result<T>`, mantém `If-Match`/`ETag` (`D_KEEP_ETAG_CONCURRENCY`) |
| Cadastros | Próprios (CRUD + CSV) | Do Vendas/ERP, réplica local só-leitura (`D_MASTER_DATA_OWNED_BY_VENDAS`, `D_MASTER_DATA_LOCAL_REPLICA`) |
| Gerenciador | pnpm 10 (monorepo) | `dotnet` (back) · Yarn 1.22 (front) |
| CI/CD | GitHub Actions (lint, tipos, testes, build, smoke docker) | Azure DevOps + ECR, dev/homolog/prod (`D_DEPLOY_LIKE_VENDAS`) |
| Testes | Vitest: ~570 casos API + ~40 front, tetos de desempenho | xUnit + Moq nos handlers (`D_TESTS_BOILERPLATE_PATTERN`) |
| Mantenedor | — | Time interno GMill (`D_MAINTAINED_BY_GMILL_TEAM`) |
| Fase | — | Back primeiro, depois telas (`D_PHASE_BACK_FIRST`) |

## Camadas

### 01 — Núcleo
- [Project charter](01-core/project-charter.md) — visão, escopo, restrições, fases
- ADRs:
  - [001 — Back como microsserviço .NET 10 próprio](01-core/adr/001-back-own-dotnet-microservice.md)
  - [002 — Front como feature do Vendas Front](01-core/adr/002-front-as-vendas-front-feature.md)
  - [003 — Identidade pelo token do Vendas](01-core/adr/003-auth-vendas-token.md)
  - [004 — Cadastros do Vendas/ERP em réplica local](01-core/adr/004-master-data-local-replica.md)
  - [005 — Postgres próprio e port do motor SQL](01-core/adr/005-own-postgres-sql-port.md)
  - [006 — Contrato HTTP: `Result<T>`, `vendas/api/`, `If-Match`/`ETag`](01-core/adr/006-http-contract-envelope-etag.md)
  - [007 — CSV só de carteiras e vínculos, em BackgroundService](01-core/adr/007-csv-portfolios-links-background-job.md)
  - [008 — Outbox de vínculos com entrega segura](01-core/adr/008-link-outbox-safe-delivery.md)
  - [009 — Testes no padrão do boilerplate](01-core/adr/009-tests-boilerplate-pattern.md)
  - [010 — Correções de segurança sobre o boilerplate](01-core/adr/010-boilerplate-security-fixes.md)

### 02 — AI-Context
- [AI development guide](02-ai-context/ai-development-guide.md) — padrões do alvo, gotchas, segurança
- [Codebase guide](02-ai-context/codebase-guide.md) — mapa AS-IS, estrutura TO-BE e mapa "de → para"

### 03 — Domínio
- [Business logic](03-domain/business-logic.md) — carteira, elegibilidade, disputa, distribuição, vínculos, visibilidade, CSV
- [API specification](03-domain/api-specification.md) — endpoints AS-IS, erros, e o contrato TO-BE

### 04 — Workflow
- [Contributing](04-workflow/contributing.md) — branches, setup, build, testes, deploy
- [Troubleshooting](04-workflow/troubleshooting.md) — problemas conhecidos do AS-IS e armadilhas do port
- [Architecture challenges](04-workflow/architecture-challenges.md) — riscos da migração e melhorias

## Contextos relacionados
- Negócio: [`docs/business-context/`](../business-context/index.md) (stub — rodar `/docs:build-business-docs`)
- Compliance: [`docs/compliance-context/`](../compliance-context/index.md) (stub — LGPD: rodar `/docs:build-compliance-docs`)

## Fontes consultadas
- Código e config deste repo (`apps/api`, `apps/web`, `packages/shared`, `docker/`, `compose*.yaml`, `scripts/`, `.github/workflows/ci.yml`), commit `0b0b2cf`.
- Boilerplate back: `~/boilerplates/gmill/meugmill/back` (`BOILERPLATE.md`, `src/`, `tests/`) — snapshot local, sem git.
- Boilerplate front: `~/boilerplates/gmill/meugmill/front/front.md` (baseado no `MeuGmillProdutoFront`, snapshot de 2026-07-23).
- Decisões do maestro: 14 nós `D_*` selados em 2026-10-14.

## Pendências de validação

| Item | Onde impacta | Estado |
|---|---|---|
| Nomes das claims de perfil e filial no JWT do Vendas, e algoritmo/chave de assinatura | ADR 003, visibilidade | `[TO BE COMPLETED]` |
| Orquestrador de deploy (ECS/K8s/VM) e nomes dos ambientes | ADR 001, contributing | `[TO BE COMPLETED]` |
| Mecanismo de sincronização dos cadastros (evento, job, view, CDC) e contrato de carga | ADR 004 | `[TO BE COMPLETED]` |
| Nomes canônicos dos perfis (`admin`, `supervisao`, `gestor`, `vendedor`) | visibilidade | `[INFERIDO]` no legado |
| Origens CORS de homolog/prod | contrato HTTP | `[TO BE COMPLETED]` |
| Divergência .NET 9 (`front.md`) × .NET 10 (boilerplate back) | ADR 001 | registrada; alvo = .NET 10 |
| Divergência do envelope: `DataRequisicao`/`IList<string>` (back) × `requestDate`/`unknown[]` (front) | ADR 006 | registrada; decidir na implementação |
