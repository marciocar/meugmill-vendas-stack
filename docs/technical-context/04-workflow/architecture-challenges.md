# Architecture Challenges — migração para o padrão GMill

> **Última Atualização:** 2026-10-14 · riscos ordenados por impacto. Cada um aponta o ADR e a mitigação sugerida.

| # | Desafio | Impacto | ADR | Mitigação |
|---|---|---|---|---|
| C1 | **Motor SQL sem teste contra Postgres.** Elegibilidade, disputa, visibilidade e distribuição são SQL cru reescrito para outro dialeto, e o padrão de testes escolhido não usa banco real | Alto — regra de negócio errada passa despercebida | 009, 005 | (a) smoke E2E equivalente ao `scripts/smoke.sh` contra homolog; (b) rodar o mesmo seed (`scripts/demo/seed.mjs`) nos dois backs e comparar prévias/disputas/distribuições; (c) reavaliar Testcontainers só para o motor se (a)/(b) acharem divergência |
| C2 | **Perda da serialização do SQLite.** Finalize, tomada de vínculos, distribute, troca de filial e CSV assumem 1 escritor | Alto — vínculo duplicado, disputa inconsistente | 005 | lock por filial (`pg_advisory_xact_lock`) em toda escrita que mexe em vínculos/atribuições; índice único parcial continua como rede |
| C3 | **Outbox no Postgres pula eventos** (sequence ≠ ordem de commit) | Médio agora (sem consumidor), alto quando houver | 008 | snapshot xmin ou sequência por filial sob lock |
| C4 | **Claims do token Vendas desconhecidas** | Bloqueia a fase 1 | 003 | levantar nomes de claims, algoritmo e chave com o time do Vendas |
| C5 | **Sincronização dos cadastros** (mecanismo, frequência, inativações → encerramento de vínculos com evento) | Alto | 004 | contrato de carga; sincronizador aplica as mesmas regras de `endLinksWhere` |
| C6 | **Simulação de CSV sem cópia do banco** | Médio — transação longa segura locks | 007 | blocos de 200 linhas; medir; considerar dry-run em schema temporário se travar |
| C7 | **Desempenho no Postgres** (tetos de 1,5 s / 5 s / 300 ms medidos em SQLite local) | Médio | 005 | portar índices compostos; medir com o seed de volume; cache por filial para 100 carteiras (já previsto) |
| C8 | **Envelope divergente entre os boilerplates** (`DataRequisicao` × `requestDate`, `IList<string>` × `unknown[]`) e `Result` sem 403/409/428 | Médio — front não reconhece erros | 006 | seguir o que a API Vendas serializa hoje; estender `Result`/filtro |
| C9 | **Normalização Unicode JS × .NET** | Médio — nomes de carteira colidem diferente | — | função própria + tabela de casos do legado (`portfolios.test.ts`, `normalize`) |
| C10 | **Passivos de segurança do boilerplate** | Alto se copiados | 010 | checklist do ADR 010 na criação do repo; sinal ao mantenedor do boilerplate |
| C11 | **Orquestrador e ambientes indefinidos** | Baixo agora | 001 | confirmar com a infra GMill |
| C12 | **Wizard sem abas montadas** (dirty-guard, preservação de etapa) | Baixo | 002 | `useBlocker` + estado do rascunho na URL/Zustand |

## Melhorias desejadas (fora do port 1:1)
- Cache por filial do resultado da disputa (já previsto no legado para 100 carteiras).
- Pusher para fim de job de CSV no lugar do polling (exige evento no back).
- Consumidor da outbox (Vendas/ERP/BI) quando houver demanda.
