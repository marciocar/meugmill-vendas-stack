# Business Logic — Carteira de Clientes

> **Última Atualização:** 2026-10-14 · fonte: código AS-IS (`apps/api/src/domain/`, `packages/shared/src/`) e testes
> `apps/api/test/domain/`. Caminhos curtos abaixo são relativos a `apps/api/src/`.
> **Esta é a especificação de comportamento a preservar no port .NET.** Onde o alvo muda a regra, está marcado **TO-BE**.

## Glossário

| Termo (pt-BR) | Identificador | O que é |
|---|---|---|
| Filial | `branch` | Unidade da GMill. O token traz **códigos** de filial (`branches.code`), não ids |
| Carteira | `portfolio` | Conjunto de clientes de uma filial, definido por filtros + ajustes, distribuído entre vendedores |
| Responsável | `responsibleSub` | `sub` do gestor dono da carteira |
| Subgrupo de produto | `product_subgroup` | Eixo da distribuição: um vendedor por cliente × subgrupo |
| Ajuste manual | `override` (`include`/`exclude`) | Inclusão/exclusão explícita de cliente |
| Disputa | conflict | Quando o mesmo cliente casa em mais de uma carteira da filial |
| Atribuição | `assignment` | Rascunho: cliente × subgrupo → vendedor |
| Vínculo | `link` | Atribuição efetivada pela finalização; histórico, nunca apagado |
| Evento de vínculo | `link_event` | Outbox `created`/`ended` |

## Identidade e escopo
- `Actor = { sub, roles, branchCodes }` (`packages/shared/src/index.ts`, `domain/shared/authz.ts`).
- O escopo resolve filiais **ativas e inativas** do token (`authz.ts` `resolveScopeBranches`).
- Fora do escopo → **404** (não revela existência). Criar/mover para filial fora do escopo → 403 (`assertAllInScope`).
- **TO-BE:** os perfis e as filiais vêm das claims do token Vendas (ADR 003).

## Auditoria e concorrência (todas as tabelas de domínio)
`active`, `deactivated_at`, `version` (começa em 1), `created_at`/`updated_at` (epoch ms), `created_by`/`updated_by`
(`sub`) — `db/schema.ts:22-30`. Escrita versionada sem `If-Match` → 428; versão errada → 409 `version_conflict`
(`shared/audit.ts`). Operação que não muda nada é **no-op e não sobe a versão**.

## Cadastros (AS-IS) → réplica (TO-BE)
- **Localidades IBGE:** 27 UFs, 5.571 municípios, imutáveis, cache em memória (`geo/service.ts`). Ficam como seed no alvo.
- **Catálogos** (subgrupo, rede, grupo econômico, tipo de carteira): `code` + `name`, globais (sem filial).
- **Vendedor:** `code`, `name`, `user_sub` opcional e único quando presente; N:N com filiais (`seller_branches`) com `active` próprio.
- **Cliente:** CNPJ global único (numérico ou alfanumérico), razão/fantasia com `*_key`, UF + município + bairro (`neighborhood_key`),
  rede e grupo opcionais; N:N com filiais (`customer_branches`) com `active` próprio.
- **Estado efetivo** para o ator = registro global ativo **E** ≥ 1 vínculo ativo no escopo (`shared/links.ts`).
- **TO-BE:** tudo isto é réplica do Vendas/ERP (ADR 004). As regras de CRUD, "cobrir todas as filiais para alterar dado
  compartilhado" e `planLinkChange` deixam de existir na carteira, **mas** os efeitos de inativação sobre vínculos permanecem (ver §Vínculos).

## Carteira
Campos: `branch_id`, `name` + `name_key` (único por filial), `description`, `responsible_sub`, `portfolio_type_id`,
`status` (`draft|active`), `finalized_at/by` + auditoria (`db/schema.ts:188-219`).

**Dois eixos independentes:** `status` (ciclo do wizard) × `active` (inativação).

| Evento | Efeito |
|---|---|
| criar | `draft`, v1, ativa |
| finalizar | `status=active`, `finalized_at` |
| inativar | encerra todos os vínculos + volta a `draft` (`portfolios/service.ts:146-149`) |
| trocar de filial | encerra todos os vínculos + `draft`; ajustes sem vínculo na nova filial → 400 com rollback |
| reativar | **não** recria vínculos (precisa finalizar de novo); filial ou tipo inativo → 400 |

Filhos: regiões (`state|municipality|neighborhood`, chave única por carteira), redes, grupos, vendedores por subgrupo
(`portfolio_sellers`), ajustes (`include|exclude`, mutuamente exclusivos), atribuições (um vendedor por célula).

**Limites:** 500 regiões · 200 redes · 200 grupos · 500 pares vendedor×subgrupo · 5.000 ajustes · 5.000 itens por
chamada de atribuição · nome 120 · descrição 1.000.

## Elegibilidade (prévia)
Motor em `eligibility/query.ts:134-210`, uma única construção SQL (`membersSql`):

1. Cada critério gera linhas `(carteira, cliente, bit)`: **1** bairro · **2** município · **4** UF · **8** rede · **16** grupo · **32** inclusão · **64** exclusão. `SUM(bit)` por par = OU.
2. Só entra cliente **válido**: `active` **e** vínculo ativo com a filial da carteira.
3. Casamento de região: bairro = mesmo `municipality_code` **e** `neighborhood_key`; município = `municipality_code`; UF = `state_code`.
4. `rank` = nível mais específico que casou: bairro 3 · município 2 · UF 1.
5. **OU dentro do tipo de critério, E entre os tipos:**
   ```
   is_filter = (has_reg or has_net or has_grp)
           and (not has_reg or rank is not null)
           and (not has_net or by_net)
           and (not has_grp or by_grp)
   ```
   Carteira sem nenhum critério nunca casa por filtro.
6. **Posto na disputa** (`eff_rank`):
   ```
   inclusão                      → 6
   is_filter e não excluído      → max(grupo?5, rede?4, rank)
   senão                         → NULL (não concorre)
   ```
7. Membro da prévia = `eff_rank is not null`. Filtro `source`: `filter` (`is_filter`) ou `manual`.
8. Busca `q`: `LIKE` sobre `legal_name_key`/`trade_name_key` normalizados ou CNPJ normalizado, com `%`/`_` escapados.

**Ajustes manuais** (`eligibility/service.ts:142-177`): substituição total; ≤ 5.000; sem repetição nem interseção;
todo cliente precisa ter vínculo (ativo ou não) com a filial da carteira, senão "Cliente inválido ou fora do escopo"
(inexistente e fora da filial são indistinguíveis); inclusão exige membro válido. Ajustes órfãos (cliente sem vínculo
com a filial) são omitidos e não contam.

## Disputa entre carteiras
`eligibility/query.ts:212-278`.
- Concorrentes de P: mesma filial, `active`, id ≠ P. **Rascunhos e ativas concorrem**; inativas e de outra filial não.
- Resolução por cliente: `assigned` se nenhum concorrente ou `eff_rank > cmax`; `lost` se `<`; **`blocked` se empate**.
- Força: **manual 6 > grupo 5 > rede 4 > bairro 3 > município 2 > UF 1**. Dois manuais empatam → bloqueado nas duas.
- Membros efetivos = só `assigned`, em ordem de id. **Sem cache:** recalculado a cada chamada.
- Contagens de conflito no agregado só com `include=conflicts` (custoso).

## Distribuição
`distribution/grid.ts`, `service.ts`, `strategies/balanced.ts`.
- **Grade** = membros efetivos × subgrupos distintos de `portfolio_sellers`.
- Vendedor **utilizável** = ativo **e** vínculo ativo com a filial da carteira.
- Célula válida = cliente ∈ membros · subgrupo ∈ carteira · vendedor ∈ utilizáveis do subgrupo.
- Status da célula (derivado na leitura, nunca limpo em background): `unassigned` · `assigned` · `stale` (gravada mas inválida).
- `distribute`: preenche só `unassigned`/`stale` da grade; preserva as válidas; subgrupo sem vendedor utilizável → `skippedSubgroupIds`; só sobe versão se gravar.
- **`balanced`:** min-heap por (contagem válida existente + atribuídas na passada); desempate pelo **menor código** do vendedor em
  comparação **ordinal** (UTF-16); clientes em ordem de id → **determinística**. Estratégia plugável (`DistributionStrategy`).
- `replaceAssignments`: `set`/`clear` atômicos, ≤ 5.000 itens, sem célula repetida, erro `set[i]: motivo` sem eco de dado.

## Vínculos e finalização
`links/service.ts:163-271`, `links/write.ts`. Ordem de `finalize`:
1. checagens de edição (ver §Autorização);
2. filial inativa → 400;
3. `blocked > 0` → 409 `portfolio_has_conflicts {blocked}`;
4. célula da grade `unassigned`/`stale` → 409 `portfolio_incomplete {unassigned, stale}` (stale fora da grade é ignorada);
5. diff contra os vínculos ativos de P: mantém se vendedor igual **e** mesma filial; senão encerra; cria os que faltam;
6. nada a fazer e já `active` → no-op (versão não muda);
7. **tomada:** vínculos ativos de outras carteiras nas células a criar são encerrados e a versão delas sobe;
   rede fail-closed `stillWinning` → se a outra ainda vence, 409 `link_conflict {portfolioIds}`;
8. **encerra antes de criar** (índice único parcial); `status=active`, `finalizedAt`.

**Invariante:** no máximo **1 vínculo ativo por (filial, cliente, subgrupo)** em todas as carteiras da filial
(índice único parcial `WHERE active`). Vínculo **nunca é apagado**: encerrar = `active=false` + `valid_to` + `ended_by`.

**Gatilhos de encerramento fora da carteira** (sempre com evento): vendedor inativado (global → todos os vínculos; por filial →
só naquela filial); cliente idem; filial inativada → todos os vínculos da filial; carteira inativada; carteira que troca de filial.
**TO-BE:** os gatilhos de vendedor/cliente/filial passam a ser aplicados pelo **sincronizador da réplica** (ADR 004).

**Outbox:** evento `ended` gravado antes do update; `created` na mesma transação. Leitura exige leitura ampla (senão 403),
`branchId` fora do escopo → 404, padrão 100, máx. 1.000. **TO-BE:** entrega segura no Postgres (ADR 008).

## Visibilidade e autorização
`visibility/profiles.ts`, `visibility/sql.ts`, `portfolios/access.ts`, `portfolios/authz.ts`.

| Perfil `[INFERIDO]` | Pode |
|---|---|
| `admin` | escrever; leitura ampla na filial |
| `supervisao` | leitura ampla, **nunca escreve** (nem como responsável) |
| `gestor` | ver carteiras em que é responsável e seus clientes vinculados |
| `vendedor` | ver os clientes dos vínculos onde `sellers.user_sub = sub` |
| sem perfil conhecido | `VISIBILITY_LEGACY=allow` → leitura ampla; `deny` → nada amplo. **TO-BE prod: `deny`** |

- Papel quase conhecido (`"Admin"`, `" admin"`, `"VENDEDOR"`) → 403 em toda operação (`withRoleGuard`, log `role_case_mismatch`). **TO-BE:** behavior de pipeline do Mediator.
- Clientes visíveis = união: amplo (qualquer vínculo com filial do token) ∪ gestor ∪ vendedor; gestor/vendedor exigem cliente ativo + vínculo ativo com a filial.
- Carteira: ler = amplo, responsável ou vendedor que atua nela; dados internos (prévia, ajustes, atribuições, vínculos) = amplo ou responsável;
  vendedor recebe agregado **reduzido**.
- Editar = admin ou responsável (não supervisão). Trocar filial/responsável, inativar, reativar = só admin. Criar = admin com a filial no token.
- **Ordem dos erros de escrita (contrato):** 404 → 403 → 403 (troca) → 428 → 409 `portfolio_inactive` → 409 `version_conflict`.

## Importação e exportação CSV
`csv/codec.ts`, `layouts.ts`, `jobs.ts`, `import/*`. **TO-BE:** só layouts de carteiras e vínculos (ADR 007).
- **Codec:** UTF-8 estrito (BOM opcional), `;`, aspas RFC 4180, CRLF/LF/CR; vazias ignoradas; ≤ 100 mil linhas, campo ≤ 4.000, ≤ 64 colunas, ≤ 16 MB.
- **Fórmula:** exportação prefixa `'` se `/^'*[=+\-@\t\r]/`; importação retira exatamente um.
- Listas com `|` (por isso códigos não aceitam `|`, `:`, espaço).
- Cabeçalho desconhecido/repetido/ausente → erro de arquivo (`invalid`); nº de campos diferente → `malformed_row`.
- Chave natural repetida → **todas** as ocorrências `duplicate_key`. Chaves: carteira = filial + `portfolioNameKey(nome)`; vínculo = filial + carteira + CNPJ + subgrupo.
- **Simulação** roda as escritas reais; erros → `invalid`, senão `validated` com `expiresAt = +24h`.
- **Confirmação** exige `validated`, arquivo em memória e **mesmo `actorScope`**; confere `targetVersion` e `action` por linha (`changed_since_validation`).
- Unidade atômica: linha (carteiras) · **carteira inteira** (vínculos); falha desfaz a unidade.
- Layout de vínculos = **conjunto completo** de cada carteira citada → diff de atribuições + `finalize`. CNPJ inexistente usa id fictício para responder igual a "não é membro".
- Região em carteira: `UF`, `UF/IBGE` ou `UF/IBGE/bairro` (bairro pode ter `/`).
- Estados do job: `validating, validated, invalid, applying, applied, partially_applied, failed, cancelled, expired, interrupted`.
- Limites: 5 jobs abertos/usuário, 48 MB/usuário, 256 MB total; só admin importa; job visível só ao criador.
- Exportação: mesmo layout e escopo das listagens; ida e volta resulta em `unchanged`.

<a id="normalizacoes"></a>
## Normalizações (replicar bit a bit no .NET)
- `searchKey`/`neighborhoodKey` = trim + colapso de espaços → **NFD** → remove `\p{M}` → **`toUpperCase()` do JS** (`shared/normalize.ts`).
  ⚠️ `ToUpperInvariant` do .NET **não** faz mapeamento completo (`ß` → `SS` no JS) e `\s` difere entre JS e .NET.
- `portfolioNameKey` = `searchKey` + toda sequência não letra/dígito vira um espaço ("Norte — Farmácias" = "NORTE/FARMÁCIAS").
  Recalculado a cada boot no legado (`refreshPortfolioNameKeys`).
- CNPJ: remove `./-` e espaço, maiúsculas; formato `[0-9A-Z]{12}[0-9]{2}`; rejeita repetido; DV com valor = ASCII − 48 e pesos 2..9 (`shared/cnpj.ts`).
- Município determina a UF; `stateCode` divergente → 400.
- `responsibleSub`/`userSub`: só trim, **sensível a caixa**.

<a id="desempenho"></a>
## Desempenho (tetos do legado, `pnpm test:perf`)
| Cenário | Volume | Teto |
|---|---|---|
| Prévia (1ª página, meio, `q+source`) | 50 mil clientes | < 1,5 s (afirmado sempre) |
| Disputa, 20 carteiras sobrepostas | 50 mil + 200 mil de outra filial | < 1,5 s |
| Disputa, 100 carteiras | 50 mil | < 7 s (medido 2,3–2,6 s; **risco aceito**, cache por filial previsto) |
| `distribute` do zero / incremental | 150 mil células | < 5 s; diferença entre vendedores ≤ 1 |
| 1ª finalização / re-finalização (1%) | 150 mil vínculos | < 5 s / < 2 s |
| "Meus clientes", `check(1000)` | 150 mil vínculos | < 300 ms |
| CSV simulação/gravação | 50 mil linhas | < 120 s cada; p99 do event loop < 150 ms |

Pontos caros: `previewPage` carrega todos os ids e faz busca binária pelo cursor; `loadAssignmentGrid` é O(membros × subgrupos);
`stillWinning` recalcula a disputa de cada carteira tomada; CSV faz várias consultas por linha (com latência de rede do Postgres, os 120 s ficam em risco).

<a id="casos-de-borda"></a>
## Casos de borda (roteiro de revisão — vêm de `apps/api/test/domain/`)
- **Disputa:** dois manuais bloqueiam nas duas; empate triplo no topo (2 bloqueadas, 1 perde); exclusão tira da disputa; inativa não concorre
  mas a prévia dela é resolvida contra as ativas; rascunho × ativa concorrem; inativar a concorrente libera o cliente na hora.
- **Elegibilidade:** bairro homônimo em outro município não casa; `q` com `%`/`_` escapado; 5.000 inclusões ok, 5.001 → 400;
  cliente só da filial CAR numa carteira SER → 400 mesmo para quem tem as duas; troca de filial com ajuste órfão → 400 e rollback total.
- **Distribuição:** `stale` por perda de disputa, par removido, vendedor inativado/sem vínculo; `clear` de `stale` idempotente;
  determinística entre bancos iguais.
- **Vínculos:** re-finalizar sem mudança não sobe versão; carteira vazia finaliza; P toma o cliente de Q mesmo com Q incompleta;
  reativar não recria; outbox com duas filiais intercaladas sem buraco nem repetição.
- **Visibilidade:** vendedor sem `userSub` não vê nada; perfis somam sem duplicar; `check` não distingue inexistente de invisível;
  supervisão nunca escreve; escrita em carteira alheia, existente ou não → mesmo 404.
- **Carteiras:** `—`, `-`, `–`, `/` colidem no nome; "São Torquato" = "SAO TORQUATO"; responsável com versão velha tentando trocar filial → **403, não 409**.
- **CSV:** carteira que toma cliente de outra no mesmo arquivo grava igual na simulação e na confirmação; apóstrofo + fórmula volta igual.
