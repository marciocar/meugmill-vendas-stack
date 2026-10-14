# Contexto Técnico — MeuGmill Vendas (Carteira de Clientes)

> **Stub de adoção (2026-10-14).** Só contém o que está declarado no `README.md` e nos `package.json`.
> Preencher via `/docs:build-tech-docs` (precedido de `/docs:reverse-consolidate`) na sessão deste repo.

## Stack & Arquitetura
- Monorepo **pnpm** (`pnpm-workspace.yaml`: `apps/*`, `packages/*`), Node **>=22**, TypeScript ~5.9.
- `apps/api` — API Node + TypeScript; contrato em `apps/api/openapi-v1.json`; migrations Drizzle em `apps/api/drizzle/`.
- `apps/web` — componente web embarcável (Vite).
- `packages/shared` — tipos e regras compartilhados (`@meugmill/shared`).
- `docker/` + `compose.yaml` — stack local (API, web, IdP de teste) presa a `127.0.0.1`.

## Comandos
- `pnpm lint && pnpm typecheck && pnpm test && pnpm build`
- `docker compose up -d && bash scripts/smoke.sh`

## Decisões (ADRs)
_(a levantar)_

## Pendências
- [ ] Arquitetura detalhada (C4), domínio e integrações — `/docs:build-tech-docs`.
