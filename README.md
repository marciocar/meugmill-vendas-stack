# MeuGmill Vendas — Carteira de Clientes

Microserviço da **carteira de clientes** do MeuGmill Vendas (Grupo GMill — distribuição farmacêutica
atacadista, Serra/ES): cadastro de carteiras, elegibilidade por filtros, conflitos, distribuição entre
vendedores, vínculos, importação CSV e as telas do wizard.

## Estrutura

| Pasta | Conteúdo |
|---|---|
| `apps/api` | API (Node 22 + TypeScript), contrato em `apps/api/openapi-v1.json` |
| `apps/web` | Componente web embarcável (wizard, CSV e "meus clientes") e manual em `/docs/` |
| `packages/shared` | Tipos e regras compartilhados |
| `docker/` | Dockerfiles, nginx e IdP de teste |
| `scripts/` | Smoke da stack e seed da demo (dados fictícios) |

## Rodar

```bash
pnpm install
pnpm lint && pnpm typecheck && pnpm test && pnpm build
```

Stack completa (API, web e IdP de teste), presa a `127.0.0.1`:

```bash
docker compose up -d
bash scripts/smoke.sh
node scripts/demo/seed.mjs   # carga da demo com dados fictícios
```

As variáveis e seus defaults estão documentados no próprio `compose.yaml`.

> O IdP de teste entrega token de administrador a quem pedir: não exponha a stack sem autenticação na frente.
