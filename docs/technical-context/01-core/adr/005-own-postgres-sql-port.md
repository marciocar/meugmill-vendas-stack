# ADR 005 — Postgres próprio e port do motor SQL

> **Status:** Aceito · **Data:** 2026-10-14 · **Última Atualização:** 2026-10-14
> **Nó KG:** `D_DB_OWN_POSTGRES`

## Contexto
O legado usa SQLite com 1 escritor (transações `BEGIN IMMEDIATE`, `busy_timeout=5000`, WAL). Várias invariantes
dependem dessa serialização. O motor de elegibilidade/disputa é SQL cru com bitmask, CTEs materializadas e
truques do dialeto SQLite. Há 13 migrations Drizzle (`apps/api/drizzle/0000`–`0012`).

## Decisão
- **Postgres próprio** da carteira, via EF Core + Npgsql (boilerplate). Sem migração de dados do SQLite (demo).
- Schema recriado em **migrations EF Core novas** (não portar as do Drizzle; a `0006` é vazia e a `0003` é backfill SQLite).
- O motor de elegibilidade/disputa/visibilidade continua **SQL cru** (`FromSqlRaw`/`SqlQuery` ou Dapper) reescrito para Postgres.
- Escritas que dependem de serialização tomam **lock por filial** (`pg_advisory_xact_lock(branch_id)` ou
  `SELECT … FOR UPDATE` na linha da filial) dentro da transação.

## Tradução obrigatória SQLite → Postgres
| SQLite (legado) | Postgres (alvo) |
|---|---|
| epoch ms em `integer` | `bigint` (ou `timestamptz` com conversão explícita) |
| booleano 0/1, `active = 1`, `sum(kind='include')` | `boolean`, `active`, `count(*) filter (where …)` |
| `case when code & 1 then` | `case when (code & 1) <> 0 then` |
| `json_each(?)` / `json_extract` | `unnest(@ids)` / `= ANY(@arr)` |
| `max(a,b,c)` escalar | `GREATEST(a,b,c)` |
| `select null … where 0` | `where false` |
| `LIKE` case-insensitive ASCII | comparar só `*_key` já maiúsculas; `ILIKE` fora delas |
| colação binária (`order by code`) | `COLLATE "C"` + `StringComparer.Ordinal` |
| `SQLITE_CONSTRAINT_UNIQUE` | `23505` (`PostgresException.SqlState`) |
| `AUTOINCREMENT`, JSON em texto | `identity`, `jsonb` |
| coluna gerada `region_key` (`int || text`) | `GENERATED ALWAYS AS (… ::text) STORED` ou único `NULLS NOT DISTINCT` (PG 15+) |
| `count(*) from (subquery)` | alias obrigatório (PG < 16) |
| `id > max(id)` lido antes para achar eventos criados | `INSERT … RETURNING id` |

## Consequências
- ✅ Concorrência real (várias instâncias da API), backups e operação padrão.
- ⚠️ O desempenho do motor precisa ser **remedido** em Postgres; os tetos do legado são referência, não garantia.
- ⚠️ Os testes que afirmam plano de execução (`EXPLAIN QUERY PLAN`, "sem TEMP B-TREE") não portam.
- ⚠️ `BOILERPLATE.md` §6: fixar `HasColumnType` explícito para não deixar convenção do Npgsql mudar schema.
