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
| **agremiação (Institution)** | Club, University | `institutions` (novo) | Novo módulo; `Institution` é a entidade; `Club`/`University` são tipos; relação futura com times/atletas via `institution_id`; afiliação a `organizations` é mutável |

> Nomenclatura: **Institution** (entidade) para evitar colisão `Club` (entidade) vs `Club` (tipo). Tabela sugerida `institutions`. Validar com ADR-003/007 antes de migrar.

### 2. Afiliação

- `Institution` pertence a uma `Organization` (`institutions.organization_id` FK).
- **Afiliação é mutável**: instituição pode mudar de organização.
- Transferência deve ser auditada (ex: `institution_id`, `from_organization_id`, `to_organization_id`, `changed_at`, `changed_by`).

### 3. Perfis e permissões (criação)

| Perfil | `organizations` (fed/assoc/liga) | `institutions` (Club/University) |
|---|---|---|
| **ORGANIZER** | ✅ cria/edita | ✅ cria/edita |
| **MANAGER** | ❌ somente leitura | ✅ cria/edita |

- Leitura: ambos os perfis leem os dois módulos.
- Escrita: conforme tabela acima; demais roles (`ADMIN`, `ADMIN_LIGA`, `REFEREE`, `MESA`) mantêm regras atuais.
- `PENDING` continua read-only global (ADR-008).

### 4. Regras de implementação (quando for executar)

- `OrganizationType = FEDERATION | ASSOCIATION | LEAGUE`.
- `InstitutionType = CLUB | UNIVERSITY` (enum da entidade `Institution`).
- Migration para `institutions` (`id`, `name`, `type`, `organization_id` FK, `status`, timestamps); **não** reaproveitar `organizations`.
- `SecurityExpressions`: `ORGANIZATION_WRITE = "hasAuthority('STATUS_ACTIVE') and hasAnyRole('ORGANIZER','ADMIN','ADMIN_LIGA')"` e `INSTITUTION_WRITE = "hasAuthority('STATUS_ACTIVE') and hasAnyRole('ORGANIZER','MANAGER','ADMIN','ADMIN_LIGA')"`.
- Frontend: separar navegação/cards "Organizações" vs "Instituições" (label PT: Agremiações).

## Consequências
- Clareza de domínio e de permissões.
- Migração de dados necessária se houver clubes/universidades já em `organizations`.

## Não-escopo deste ADR
Implementação, migrations, APIs e telas — aguardar demais pontos do PO.

## Referências
- ADR-003 (club entity hierarchy)
- ADR-008 (PENDING read-only)
- `architecture/roles-and-permissions.md`
