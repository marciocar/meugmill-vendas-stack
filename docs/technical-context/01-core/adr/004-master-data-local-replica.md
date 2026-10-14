# ADR 004 — Cadastros do Vendas/ERP em réplica local

> **Status:** Aceito · **Data:** 2026-10-14 · **Última Atualização:** 2026-10-14
> **Nós KG:** `D_MASTER_DATA_OWNED_BY_VENDAS`, `D_MASTER_DATA_LOCAL_REPLICA`

## Contexto
Hoje a carteira é dona dos cadastros: filiais, catálogos (subgrupo de produto, rede de varejo, grupo econômico,
tipo de carteira), vendedores e clientes, com CRUD, vínculos N:N com filial (`seller_branches`,
`customer_branches`) e CSV. O motor de elegibilidade e disputa (`apps/api/src/domain/eligibility/query.ts`) é
**SQL sobre essas tabelas**, com tetos de 1,5 s para 50 mil clientes.

## Decisão
1. O **Vendas/ERP é o dono** de clientes, vendedores, filiais e catálogos. A carteira só mantém carteiras,
   filtros, ajustes, atribuições, vínculos e eventos.
2. Os cadastros chegam como **réplica local somente-leitura sincronizada** no Postgres da carteira, preservando
   as colunas que o motor usa (`active`, `municipality_code`, `state_code`, `neighborhood_key`,
   `retail_network_id`, `economic_group_id`, `legal_name_key`, `trade_name_key`, `user_sub`, vínculos por filial).
3. A tabela de localidades IBGE (27 UFs, 5.571 municípios) continua como seed próprio.

## Alternativas descartadas
- **Carteira continua dona:** duplicação e divergência com o ERP.
- **Híbrido:** dois modelos de origem.
- **Consulta on-line:** o motor SQL não roda sobre dados remotos.

## Consequências
- ✅ Saem do escopo: rotas CRUD de `branches`, `sellers`, `customers`, catálogos e `portfolio-types`; layouts CSV de cadastro;
  serviços `domain/{branches,catalog,customers,sellers,portfolio-types}` (viram leitores da réplica).
- ⚠️ **Os gatilhos de encerramento de vínculo mudam de origem.** Hoje inativar vendedor/cliente/filial encerra
  vínculos na mesma transação (`links/write.ts` `endLinksWhere`). Na réplica, o **sincronizador** precisa aplicar as
  mesmas regras (com eventos na outbox) ao receber inativações.
- ⚠️ Normalizações (`*_key`) precisam ser calculadas **na carga**, com a mesma função do legado (ver
  [business-logic §Normalizações](../../03-domain/business-logic.md#normalizacoes)).
- `[TO BE COMPLETED]` Mecanismo (evento, job periódico, view, CDC), frequência e contrato de carga com o Vendas/ERP.
