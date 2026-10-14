# ADR 008 — Outbox de vínculos com entrega segura

> **Status:** Aceito · **Data:** 2026-10-14 · **Última Atualização:** 2026-10-14
> **Nó KG:** `D_LINK_OUTBOX_KEPT`

## Contexto
Cada criação/encerramento de vínculo grava um evento em `portfolio_link_events` (`created|ended`) na mesma transação.
`GET /v1/link-events?branchId&after&limit` lê por cursor de id (`e.id > after`). No SQLite isso é seguro porque há
um escritor só. No Postgres, valores de sequence **não seguem a ordem de commit**: um consumidor que leu `after=100`
pode nunca ver o evento 99 que comitou depois.

## Decisão
Manter a outbox (consumidor futuro, ainda não definido), com **entrega segura no Postgres**. Opções em ordem de preferência:
1. Lock por filial na escrita (ADR 005) **e** leitura limitada a eventos com `xmin` anterior ao snapshot mínimo
   ativo (`pg_snapshot_xmin(pg_current_snapshot())`), garantindo que não há transação aberta com id menor.
2. Coluna de sequência atribuída sob o lock por filial (sequência por filial gravada na mesma transação), lida por cursor.
3. Publicação por relay (job lê eventos não publicados e marca `published_at`) — necessária se entrar broker.

## Consequências
- ✅ Contrato do endpoint pode continuar o mesmo (`items, nextAfter, hasMore`).
- ⚠️ Exige teste de concorrência real — que **não** existe no padrão de testes escolhido (ADR 009). Ver [architecture-challenges](../../04-workflow/architecture-challenges.md).
- `[TO BE COMPLETED]` Consumidor, SLA e se entra Pusher/broker.
