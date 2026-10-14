# ADR 009 — Testes no padrão do boilerplate

> **Status:** Aceito · **Data:** 2026-10-14 · **Última Atualização:** 2026-10-14
> **Nó KG:** `D_TESTS_BOILERPLATE_PATTERN`

## Contexto
O legado tem ~570 casos Vitest na API (357 de domínio, ~180 de rota, 80 de infra) com banco SQLite real em memória,
mais suíte de desempenho (`pnpm test:perf`, `PERF_ASSERT=1`). O boilerplate back prescreve xUnit + Moq, sem
FluentAssertions 8+, com mapper Mapster real (`BOILERPLATE.md` §7).

## Decisão
Os testes do back novo seguem **só o padrão do boilerplate**: `tests/Core.Tests`, xUnit + Moq nos handlers
(caminho feliz + not-found/erro), pipeline behaviors e `IRegister` de mapeamento. Os casos do legado **não** são
portados como suíte contra Postgres.

## Alternativas descartadas
- Portar os casos como especificação contra Postgres real (Testcontainers) — recomendada na rodada, não escolhida.
- Meta de cobertura numérica.

## Consequências
- ✅ Suíte rápida, sem Docker no CI.
- ⚠️ **Risco aceito:** o SQL cru do motor (elegibilidade, disputa, visibilidade, distribuição) e a entrega da outbox
  ficam **sem teste contra o banco real**; erros de dialeto (booleano, colação, `GREATEST`, arrays) só aparecem em
  execução. Mitigações sugeridas em [architecture-challenges](../../04-workflow/architecture-challenges.md).
- Os casos do legado continuam úteis como **roteiro de revisão** e de smoke manual (listados em
  [business-logic §Casos de borda](../../03-domain/business-logic.md#casos-de-borda)).
