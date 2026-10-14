# ADR 010 — Correções de segurança sobre o boilerplate

> **Status:** Aceito (derivado de `BOILERPLATE.md` §5, não de escolha nova) · **Data:** 2026-10-14 · **Última Atualização:** 2026-10-14

## Contexto
O próprio `BOILERPLATE.md` §5 manda: nunca hardcodar chave/connection string/senha, `appsettings.*` sem host/senha
reais, auditar histórico de git, e desligar `Validate*` do JWT só por decisão consciente. O snapshot do boilerplate
(`~/boilerplates/gmill/meugmill/back`, lido em 2026-10-14) **viola** esse checklist:

| # | Achado | Onde |
|---|---|---|
| S1 | JWT com `ValidateIssuerSigningKey`, `ValidateLifetime`, `ValidateIssuer`, `ValidateAudience` = **false** — aceita token forjado e vencido | `src/Web/Extension/JwtSecurityExtension.cs` |
| S2 | Host interno e senha de banco no `appsettings.json` versionável | `src/Web/appsettings.json` (`CONNECTIONDBHOST`, `CONNECTIONDBPASSWORD`) |
| S3 | Chave privada RSA (`"D"`) em arquivo | `src/Web/tempkey.jwk` |
| S4 | Token com validade de ~3,8 anos | `TOKENSECONDS=120000000` |
| S5 | Artefatos de execução no snapshot | `src/Web/logs/log.txt`, `src/Web/healthchecksdb` |
| S6 | `pusherSecret` e chave de mapas no pipeline do front; secret no bundle (`VITE_PUSHER_SECRET`) | `front.md` §12.2 |

## Decisão
Ao criar a solução a partir do boilerplate:
1. Validação JWT completa (todos os `Validate*` = true, `ClockSkew` explícito) — ver ADR 003.
2. `appsettings*.json` com valores **em branco**; reais só em env var / `dotnet user-secrets` / cofre do pipeline.
3. **Não copiar** `tempkey.jwk`, `logs/`, `healthchecksdb`; incluí-los no `.gitignore`.
4. Sem segredo no bundle do front (só `VITE_PUSHER_KEY`/`CLUSTER`).
5. Manter as proteções do legado que o boilerplate não tem: allowlist de algoritmos assimétricos, fail-closed
   (503 quando não dá para validar), redact de logs (token, CNPJ, SQL), 404 para fora do escopo.

## Consequências
- ✅ O serviço nasce sem os passivos do snapshot.
- 📤 **Encaminhamento (dono: quem mantém o boilerplate GMill):** S1–S6 valem para todo projeto criado a partir dele;
  rotacionar a senha de banco e a chave expostas. Esta sessão não altera o boilerplate.
