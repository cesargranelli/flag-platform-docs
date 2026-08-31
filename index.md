---
layout: home
---

# Documentação — Flag Platform

Índice da documentação técnica e de produto do monorepo.

## Arquitetura e fluxos

| Documento | Conteúdo |
|-----------|----------|
| [Visão geral da solução](architecture/overview.md) | O que é, arquitetura em camadas, componentes, fluxo ponta a ponta |
| [Mapa de componentes](architecture/components.md) | Catálogo do backend, frontend, infra e CI/CD |
| [Fluxos lógicos](architecture/logical-flows.md) | Diagramas Mermaid: conta/auth, cadastro, operação ao vivo, classificação, navegação, endpoints |
| [Papéis e autorização](architecture/roles-and-permissions.md) | Contexto de uso, roles, matriz de permissões, jornadas, segurança |

## Apps

| Documento | Conteúdo |
|-----------|----------|
| [Fluxo de telas — Referee App](apps/referee-app/fluxo-de-telas.html) | Navegação completa do app da mesa: rotas, autenticação, operação de partida e check-in |

## Produto

| Documento | Conteúdo |
|-----------|----------|
| [Visão de produto](product/vision.md) | Problema, solução, aplicações, roadmap |

## Decisões de arquitetura (ADRs)

| Documento | Decisão |
|-----------|---------|
| [ADR-001 — Filosofia do Projeto](adr/ADR-001%20-%20Filosofia%20do%20Projeto.md) | Simplicidade e velocidade sobre arquitetura antecipada |
| [ADR-002 — Monorepo](adr/ADR-002-monorepo.md) | Backend, frontend, infra e docs no mesmo repositório |
| [ADR-003 — Modular Monolith](adr/ADR-003-modular-monolith.md) | Backend monolito modular Spring Boot; sem microsserviços/K8s |
| [ADR-004 — API First](adr/ADR-004-api-first.md) | Domínio → Banco → Service → API → Flutter; REST `/api/v1` |
| [ADR-005 — Staging efêmero E2E](adr/ADR-005-staging-efemero-e2e.md) | Ambientes temporários para testes ponta a ponta |
| [ADR-006 — Team/Roster/Season](adr/ADR-006-team-roster-season-refactor.md) | Refatoração estrutural: time como entidade, elenco por temporada |

## Design

| Documento | Conteúdo |
|-----------|----------|
| [Design tokens](design/tokens.md) | Fonte da verdade visual: cores Shifty, tipografia DM Sans, formas, componentes |
| [Layout spec](design/layout-spec.md) | Especificação de layout das telas |
| [Referência Kickster](design/kickster-reference.md) | Referência de design Kickster |
| [UX — referências](design/ux/) | Specs de UX e referências de design |

## Pesquisa

| Documento | Conteúdo |
|-----------|----------|
| [Mapeamento FlagStats](research/flagstats-mapeamento.md) | Levantamento do universo FlagStats |

## Como ler esta documentação

Comece pelo [overview](architecture/overview.md) para o contexto geral e siga para [fluxos lógicos](architecture/logical-flows.md) para os diagramas de ponta a ponta. Consulte os [ADRs](adr/) para entender as decisões de arquitetura e as issues do GitHub para o planejamento.