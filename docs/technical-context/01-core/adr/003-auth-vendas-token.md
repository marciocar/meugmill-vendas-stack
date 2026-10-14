# ADR 003 — Identidade pelo token do Vendas

> **Status:** Aceito · **Data:** 2026-10-14 · **Última Atualização:** 2026-10-14
> **Nós KG:** `D_AUTH_VENDAS_TOKEN`, `D_VISIBILITY_LEGACY_DENY_PROD`

## Contexto
Hoje (`apps/api/src/plugins/auth.ts`) a API valida JWT por OIDC: JWKS remoto, `issuer`/`audience` obrigatórios,
allowlist só assimétrica (`RS256,ES256`; `none` e `HS*` proibidos), `exp`+`sub` obrigatórios, falha de JWKS → 503
`auth_unavailable` (fail-closed). Lê só `sub`, `roles` e `branch_ids` (minimização LGPD).

No Vendas, o front faz login Azure (MSAL) ou básico e **troca** por um JWT da API Vendas (`auth/login-azure`,
`auth/login-basic`; `front.md` §10.1). O `IAuth` carrega `admin`, `agentId`, `branches`, `channels`...

## Decisão
A API da carteira valida o **JWT emitido pela API Vendas** e mapeia:
- `sub` → identidade do ator;
- perfis (`admin`, `supervisao`, `gestor`, `vendedor`) ← claim(s) do Vendas `[TO BE COMPLETED]`;
- filiais (`branchCodes`) ← claim de filiais do Vendas `[TO BE COMPLETED]`.

Os nomes das claims ficam em configuração (como `OIDC_CLAIM_ROLES`/`OIDC_CLAIM_BRANCHES` hoje).
Em produção, token sem perfil conhecido **não** tem leitura ampla (`VISIBILITY_LEGACY=deny`); `allow` só em dev/demo.

## Requisitos de validação (não negociáveis — ver ADR 010)
`ValidateIssuer`, `ValidateAudience`, `ValidateLifetime` e `ValidateIssuerSigningKey` = **true**; chave de
validação por configuração (de preferência pública/assimétrica, JWKS); `ClockSkew` explícito (hoje 30 s).
Papel com caixa divergente (`"Admin"`) continua dando 403 (`role_case_mismatch`).

## Alternativas descartadas
- **Entra ID direto:** obrigaria mapear filiais fora do token.
- **Manter OIDC genérico:** só adiaria a decisão.

## Consequências
- ✅ Mesmo login do Vendas Front; o Axios do Vendas já injeta o Bearer.
- ⚠️ **Bloqueio da fase 1:** sem o nome das claims e o algoritmo/chave do token Vendas, a autorização não fecha.
- ⚠️ Se o Vendas assinar com segredo simétrico (o boilerplate usa `TOKENSECRET`), a carteira passa a depender de
  um segredo compartilhado — registrar como risco e preferir assimétrico.
- ⚠️ `TokenManagerMiddleware` do boilerplate (lista de tokens ativos no Redis, isenta `/api/login`) não se aplica
  a um serviço que não emite token.
