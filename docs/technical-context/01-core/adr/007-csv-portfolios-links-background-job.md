# ADR 007 — CSV só de carteiras e vínculos, em BackgroundService

> **Status:** Aceito · **Data:** 2026-10-14 · **Última Atualização:** 2026-10-14
> **Nó KG:** `D_CSV_PORTFOLIOS_AND_LINKS_ONLY`

## Contexto
Hoje (`apps/api/src/domain/csv/`): fila serial em memória, arquivo só em memória, **simulação sobre cópia física
do banco** (`sqlite.backup` para arquivo temporário), confirmação com checagem de versão e de ação por linha,
savepoint por unidade, `recover` na subida, limites de 16 MB / 100 mil linhas / 5 jobs por usuário / 48 MB por
usuário / 256 MB no total, TTL de 24 h para simulações validadas.

## Decisão
- Ficam os layouts de **carteiras** e **vínculos** (simulação + confirmação) e as **exportações**. Saem os de cadastro (ADR 004).
- O processamento roda num **`BackgroundService`** com estado do job no Postgres (`import_jobs`, `import_job_lines`). Uma instância processa (as demais só enfileiram/consultam).
- A **simulação** deixa de usar cópia do banco: roda as mesmas escritas de domínio numa **transação revertida ao fim**
  (ou com savepoints por bloco), gravando o relatório numa transação separada.
- O conteúdo do arquivo nunca vai ao banco de domínio (LGPD); pode ficar em memória do processo ou em armazenamento temporário (Redis/blob) `[TO BE COMPLETED]`.

## Consequências
- ✅ Mesma regra na simulação e na confirmação (as escritas são as mesmas).
- ⚠️ Transação de simulação longa segura locks (inclusive o lock por filial do ADR 005) — processar em blocos e medir.
- ⚠️ Se o arquivo ficar só em memória, restart perde simulações abertas (`interrupted`), como hoje.
- ⚠️ Preservar: proteção contra fórmula (`'` antes de `=+-@\t\r`), UTF-8 estrito com BOM opcional, `;` como separador,
  `|` em listas, `duplicate_key` em todas as ocorrências, unidade atômica = linha (carteiras) ou carteira inteira (vínculos).
- ⚠️ Limite de upload em duas camadas (proxy e API); o proxy do Vendas precisa aceitar 16 MB.
