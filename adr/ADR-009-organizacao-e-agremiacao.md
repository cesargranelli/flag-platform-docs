# ADR-009: Organização e Agremiação — separação de domínios e perfis

## Status
Proposto — escopo rascunho, sem implementação.

## Contexto
O domínio hoje chamado "organização" agregava federações, associações, ligas, clubes e universidades. A regra de negócio foi refinada: **organização** passa a conter apenas entidades de governança; **clubes e universidades** migram para um novo módulo **agremiação**.

## Decisão

### 1. Domínios

| Módulo | Entidades | Tabela(s) sugerida(s) | Notas |
|---|---|---|---|
| **organização** | Federação, Associação, Liga | `organizations` (com `type` enum) | Mantém hierarquia existente; sem clubes/universidades |
| **agremiação** | Clube, Universidade | `affiliations` (novo) | Novo módulo; relação futura com times/atletas via `affiliation_id` |

> Nomes de tabela/coluna são proposta inicial — validar com ADR-003/007 antes de migrar.

### 2. Perfis e permissões (criação)

| Perfil | `organizations` (fed/assoc/liga) | `affiliations` (clube/universidade) |
|---|---|---|
| **ORGANIZER** | ✅ cria/edita | ✅ cria/edita |
| **MANAGER** | ❌ somente leitura | ✅ cria/edita |

- Leitura: ambos os perfis leem os dois módulos.
- Escrita: conforme tabela acima; demais roles (`ADMIN`, `ADMIN_LIGA`, `REFEREE`, `MESA`) mantêm regras atuais.
- `PENDING` continua read-only global (ADR-008).

### 3. Regras de implementação (quando for executar)

- Criar enum `OrganizationType = FEDERATION | ASSOCIATION | LEAGUE`.
- Criar enum `AffiliationType = CLUB | UNIVERSITY`.
- Migration para novo domínio `affiliations`; **não** reaproveitar `organizations` para clubes/universidades.
- `SecurityExpressions`: novas constantes `ORGANIZATION_WRITE = "hasAuthority('STATUS_ACTIVE') and hasAnyRole('ORGANIZER','ADMIN','ADMIN_LIGA')"` e `AFFILIATION_WRITE = "hasAuthority('STATUS_ACTIVE') and hasAnyRole('ORGANIZER','MANAGER','ADMIN','ADMIN_LIGA')"`.
- Frontend: separar navegação/cards "Organizações" vs "Agremiações".

## Consequências
- Clareza de domínio e de permissões.
- Migração de dados necessária se houver clubes/universidades já em `organizations`.

## Não-escopo deste ADR
Implementação, migrations, APIs e telas — aguardar demais pontos do PO.

## Referências
- ADR-003 (club entity hierarchy)
- ADR-008 (PENDING read-only)
- `architecture/roles-and-permissions.md`
