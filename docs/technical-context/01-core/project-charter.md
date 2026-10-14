# Project Charter — Carteira de Clientes (migração para o padrão GMill)

> **Última Atualização:** 2026-10-14 · fonte das decisões: nós `D_*` em `docs/onion/graph/onion-adoption.kg.yaml`

## Visão

Levar a **carteira de clientes** do MeuGmill Vendas (Grupo GMill — distribuição farmacêutica atacadista,
Serra/ES) do protótipo atual (Node/TS + SQLite + componente embarcável) para o **padrão de stack da GMill**:
um microsserviço .NET 10 próprio e telas dentro do Vendas Front, mantidos pelo time interno.

O protótipo é a **especificação executável**: as regras de elegibilidade, disputa entre carteiras,
distribuição, vínculos e visibilidade já estão implementadas e testadas, e devem ser **preservadas**.

## Objetivos

1. Microsserviço .NET 10 a partir do boilerplate GMill back, com o domínio da carteira (ADR 001).
2. Contrato `vendas/api/...` com envelope `Result<T>`, mantendo a concorrência otimista (ADR 006).
3. Autenticação pelo token do Vendas, com perfis e filiais das claims (ADR 003).
4. Cadastros (clientes, vendedores, filiais, catálogos) vindos do Vendas/ERP, em réplica local (ADR 004).
5. Telas da carteira como feature do Vendas Front (ADR 002).
6. Deploy no caminho do Vendas: Azure DevOps + ECR, dev/homolog/prod (ADR 001).

## Escopo

### Dentro
- Carteiras: cadastro, wizard (informações → filtros → vendedores → resumo → clientes), inativação/reativação.
- Elegibilidade por filtros (região UF/município/bairro, rede, grupo econômico) e ajustes manuais.
- Disputa entre carteiras da mesma filial (assigned/lost/blocked).
- Distribuição de clientes × subgrupos entre vendedores (estratégia `balanced`).
- Finalização → vínculos (histórico, nunca apagados) + outbox de eventos (ADR 008).
- Visibilidade por perfil ("meus clientes", checagem de visibilidade).
- CSV de carteiras e vínculos (simulação + confirmação) e exportações (ADR 007).
- Réplica local dos cadastros do Vendas/ERP.

### Fora
- CRUD e CSV de cadastros mestres (clientes, vendedores, filiais, catálogos) — o dono é o Vendas/ERP.
- O custom element `<gmill-carteira>` e a demo de host — deixam de existir.
- IdP de teste (`mock-oauth2-server`) em qualquer ambiente além de dev local.
- Migração de dados do SQLite (é demo).

## Fases (`D_PHASE_BACK_FIRST`)

| Fase | Entrega | Critério de saída |
|---|---|---|
| 1 — Back | Solução .NET 10, schema Postgres, réplica de cadastros, contrato `vendas/api/` | Endpoints do domínio respondendo com o comportamento do legado (ver [api-specification](../03-domain/api-specification.md)) |
| 2 — Front | Rotas `pages/authenticated/carteiras/...` no Vendas Front | Fluxos do wizard, CSV e "meus clientes" equivalentes aos do componente atual |
| 3 — Corte | Deploy em homolog/prod, desligamento do protótipo | Smoke equivalente ao `scripts/smoke.sh` verde em homolog |

Não há convivência: o front legado não fala com o back novo (prefixo e envelope mudam).

## Stakeholders

| Papel | Quem |
|---|---|
| Mantenedor do serviço | Time interno GMill (devs .NET/React) — `D_MAINTAINED_BY_GMILL_TEAM` |
| Usuários | Admin, admin de filial, gestor (responsável), supervisão, vendedor |
| Dono dos cadastros | Vendas/ERP |
| Consumidor dos eventos de vínculo | Futuro (a definir) |

## Restrições técnicas

- **LGPD:** claims mínimas (`sub`, perfis, filiais); logs sem CNPJ/token/SQL; erros sem eco de dados de cliente;
  recurso fora do escopo responde 404 (não revela existência).
- **Volume:** 50 mil clientes por filial, 100 carteiras concorrentes, 150 mil células de distribuição;
  tetos do legado em [business-logic §Desempenho](../03-domain/business-logic.md#desempenho).
- **Padrão GMill:** seguir `BOILERPLATE.md` (back) e `front.md` (front), **com as correções de segurança do ADR 010**.
- **Idioma:** código e identificadores em inglês; textos de UI, docs e mensagens em pt-BR (`CLAUDE.md`).
- **Licenciamento:** nada de AutoMapper, MediatR comercial ou FluentAssertions 8+ (`BOILERPLATE.md` §2, §7).
