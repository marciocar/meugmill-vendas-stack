# ADR 002 — Front como feature do Vendas Front

> **Status:** Aceito · **Data:** 2026-10-14 · **Última Atualização:** 2026-10-14
> **Nó KG:** `D_FRONT_IN_VENDAS_FRONT`

## Contexto
O front atual é o custom element `<gmill-carteira>` (`apps/web/src/element.tsx`): Shadow DOM, token passado pelo
host via propriedade, evento `token-expired`, abas internas sem router, estado só em `useState`/Context, CSS
próprio `gc-*`. O padrão GMill é a SPA Vendas (`front.md`): AntD 5 + Tailwind 4, TanStack Query, Zustand,
RHF + Zod, React Router 7, Axios único, MSAL.

## Decisão
As telas da carteira viram **rotas do Vendas Front** (`MeuGmillProdutoFront`):

```
src/pages/authenticated/carteiras/
├─ index.tsx                     # lista de carteiras
├─ new/index.tsx
├─ [id]/<etapa>/index.tsx        # wizard: informacoes, filtros, vendedores, resumo, clientes
└─ _components/
src/pages/authenticated/meus-clientes/index.tsx
src/pages/authenticated/carteiras-csv/index.tsx
```
Rotas em kebab-case pt-BR, protegidas por `withAnyPermission([...])`; services em `data/services/portfolios/`,
hooks em `data/hooks/`, schemas Zod em `data/schema/portfolios/`, query keys em `shared/enums/key.ts`.

## Alternativas descartadas
- **SPA própria:** duplicaria login, layout e stores.
- **Continuar embarcável:** AntD (cssinjs) e Tailwind 4 dentro do Shadow DOM exigem `StyleProvider container`,
  `getPopupContainer` e CSS no shadow root — custo alto sem host que justifique.

## Consequências
- ✅ Login MSAL, layout, permissões, notificações e Axios já prontos.
- ❌ O contrato embarcável (`manual.md` "Para quem integra") e a demo de host deixam de existir.
- ⚠️ O dirty-guard do wizard vira `useBlocker` do Router 7 (hoje as abas ficam montadas para não perder edição).
- ⚠️ O polling do job de CSV vira `refetchInterval` (Pusher é opcional, só se o back publicar o evento).
- ⚠️ Os ~40 casos de `apps/web/src/*.test.tsx` servem como roteiro de comportamento; o harness (custom element +
  `routeFetch`) não é reaproveitável.
