# Contributing — Carteira de Clientes

> **Última Atualização:** 2026-10-14

## Branches
- **AS-IS (este repo):** `main` = produção/base; adoção do Onion em `onion/adopt` → PR para `main` com **merge commit**
  (nunca squash/rebase — `onion/vendor` precisa ficar na ancestralidade). Integração sem `develop` hoje.
- **TO-BE (padrão GMill):** `develop` (dev) → `main` (produção), como o Vendas (`front.md` §13).
- Commits: Conventional Commits, prefixo em inglês, assunto/corpo em pt-BR.

## AS-IS — setup e comandos
Pré-requisitos: Node 22 (`.nvmrc`), pnpm 10.29 (`packageManager`), Docker.

```bash
pnpm install
pnpm lint && pnpm typecheck && pnpm test && pnpm build
pnpm --filter @meugmill/api test:perf          # tetos de desempenho (PERF_ASSERT=1)
pnpm --filter @meugmill/api openapi:export     # regenerar openapi-v1.json após mudar rotas
pnpm --filter @meugmill/api dev                # tsx watch
docker compose up -d && bash scripts/smoke.sh  # stack em 127.0.0.1 (api 39000, idp 39080, web 39081)
node scripts/demo/seed.mjs                     # carga fictícia; --expect-unchanged na 2ª execução
```

Variáveis da API (`apps/api/src/config.ts`):

| Var | Padrão / regra |
|---|---|
| `PORT` / `HOST` | 3000 / `0.0.0.0` |
| `LOG_LEVEL` | `info` |
| `DATABASE_PATH` | `./data/carteira.sqlite` (`:memory:` nos testes) |
| `NODE_ENV` | `development` |
| `OIDC_ISSUER`, `OIDC_AUDIENCE` | **obrigatórias** |
| `OIDC_JWKS_URI` | opcional (senão discovery) |
| `OIDC_ALLOW_INSECURE_HTTP` | `false` |
| `OIDC_ALGORITHMS` | `RS256,ES256` |
| `OIDC_CLAIM_ROLES` / `OIDC_CLAIM_BRANCHES` | `roles` / `branch_ids` |
| `OIDC_CLOCK_TOLERANCE_SECONDS` | 30 |
| `CORS_ORIGINS` | vazio (nenhuma origem) |
| `VISIBILITY_LEGACY` | `allow` |

**CI AS-IS** (`.github/workflows/ci.yml`): job `check` (install → lint → typecheck → test → build) e job `docker`
(build → up → smoke → seed 2× → down). O pre-commit do Onion roda `bash .claude/validation/lint-artifacts.sh`.

## TO-BE — back (.NET 10)
```bash
dotnet --list-sdks                         # precisa do SDK 10
dotnet restore && dotnet build
dotnet test
dotnet user-secrets init --project src/Web # segredos de dev fora do appsettings
dotnet ef migrations has-pending-model-changes --project src/Infra --startup-project src/Web
```
Configuração por "env var vence appsettings" (`SettingsConfigurationHelper`): `CONNECTIONDB*`, `REDIS*`, `TOKEN*` e as
claims do token Vendas `[TO BE COMPLETED]`. `appsettings*.json` com valores em branco (ADR 010).

## TO-BE — front (Vendas Front)
```bash
yarn install            # Yarn 1.22; não commitar package-lock.json
yarn dev                # http://localhost:5173, VITE_API_BASE_URL no .env.local
yarn lint && yarn build # limpos antes do PR
```

## Testes
- AS-IS: Vitest (API ~570 casos com SQLite real em memória; web ~40 casos com happy-dom).
- TO-BE back: xUnit + Moq nos handlers, behaviors e mapeamentos (ADR 009). Cobrir caminho feliz + not-found/erro de cada handler.
- TO-BE front: `yarn lint` + `yarn build` (o boilerplate não define suíte de testes).

## Deploy TO-BE (ADR 001)
Azure DevOps (`.ci/development.yaml` em `develop`, `.ci/production.yaml` em `main`), imagem no ECR, tag `0.0.$(Rev:r)-alpha` em dev.
Ambientes dev/homolog/prod. Orquestrador `[TO BE COMPLETED]`. Segredos só como variáveis secretas da pipeline.

## Code review
- Pré-PR: `/engineer:pre-pr` (Onion) e os gates `branch-*` (code, metaspec, testes, docs).
- Mudança de regra de domínio exige ADR ou atualização de [business-logic](../03-domain/business-logic.md).
