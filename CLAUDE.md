# MeuGmill Vendas — Carteira de Clientes

Microserviço da carteira de clientes do MeuGmill Vendas (Grupo GMill). Visão geral, estrutura e
comandos de execução: `README.md`.

## 🧅 Sistema Onion

Este repositório adota o **Onion** (framework de método executável em `.claude/`). Identidade da
adoção: `.claude/.onion-version`.

- **Entrada:** `/warm-up` (contexto) → `/onion` (orientação).
- **Guardas determinísticas:** `bash .claude/validation/lint-artifacts.sh` (também no pre-commit nativo).
- **Contagens canônicas:** `docs/onion/inventory.md` (SSOT gerada; regenere com `/meta:inventory`, nunca à mão).
- **Co-evolução com o core:** `docs/evolution/` — `inbox/` (sinais para o core) e `inbound/` (relatórios do core).

## Task Manager

Provedor ativo via `TASK_MANAGER_PROVIDER` no ambiente do processo (hoje: `none`). Configure com
`/meta:setup-integration`.

## Idioma

Chat, comentários, documentação e mensagens: **pt-BR**. Código, variáveis, nomes de arquivo e de
branch, e o prefixo Conventional dos commits: **inglês** (skill `language-standards`).

## Contextos do projeto (L1+)

- `docs/business-context/` — negócio · `docs/technical-context/` — técnico · `docs/compliance-context/` — compliance.
- As meta-specs em `docs/meta-specs/` são o **L0 do framework**; os contextos acima são a governança do projeto.

## Branches

- `main` — produção e base atual.
- Integração: sem `integration_branch` carimbado — o alvo dos PRs é resolvido a cada PR
  (`develop` se existir, senão `main`). A evolução do Onion chega por PR a partir de `onion/adopt`
  (adoção) e `chore/onion-update-<pin>` (updates); `onion/vendor` é a base do merge 3-way — não use direto.
