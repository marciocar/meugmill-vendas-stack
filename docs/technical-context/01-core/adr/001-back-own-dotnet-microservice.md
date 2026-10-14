# ADR 001 — Back como microsserviço .NET 10 próprio

> **Status:** Aceito · **Data:** 2026-10-14 · **Última Atualização:** 2026-10-14
> **Nós KG:** `D_BACK_OWN_MICROSERVICE`, `D_DEPLOY_LIKE_VENDAS`, `D_PHASE_BACK_FIRST`

## Contexto
O back atual é Node 22 + Fastify 5 + SQLite (`apps/api`). O padrão GMill é ASP.NET Core em Clean Architecture
(`~/boilerplates/gmill/meugmill/back`). A API Vendas existente (`MeuGmillProdutoApi`) está em .NET 9
segundo o `front.md` §1; o boilerplate back já foi migrado para .NET 10 (`BOILERPLATE.md` §1).

## Decisão
Criar uma **solução .NET 10 nova** a partir do boilerplate back (`Core` / `Infra` / `Web` + `tests/Core.Tests`),
dedicada à carteira, com:
- Autofac + minimal hosting (`Program.cs`), Mediator (martinothamar) com `ValidationBehavior`, Mapster `IRegister`,
  FluentValidation via pipeline, `GlobalExceptionHandler : IExceptionHandler`, Serilog, Swashbuckle + ReDoc,
  health em `/status`, SonarAnalyzer em `Directory.Build.props`.
- Deploy como o Vendas: Azure DevOps (`.ci/`), imagem no ECR, ambientes dev/homolog/prod, branches `develop`/`main`.
- Fase 1 da migração (antes das telas).

## Alternativas descartadas
- **Módulo dentro do MeuGmillProdutoApi:** acopla ao ciclo de release do Vendas e obriga a resolver o .NET 9 × 10.
- **Pesquisar a topologia antes:** atrasaria o desenho do alvo sem mudar a recomendação.

## Consequências
- ✅ Domínio isolado, sem herdar dívida da API Vendas.
- ⚠️ Infra própria (banco, pipeline, imagem) — orquestrador `[TO BE COMPLETED]` (ECS/K8s/VM).
- ⚠️ Divergência registrada: `front.md` cita ASP.NET Core 9; o alvo deste serviço é .NET 10.
- ⚠️ O boilerplate traz partes que **não** servem aqui e devem sair na criação: entidades `ParametroSistema`/`Auditoria`
  de exemplo, `MailHelper`/`BlobStorage` se não usados, `TokenManagerMiddleware` (ver ADR 003 e 010).
