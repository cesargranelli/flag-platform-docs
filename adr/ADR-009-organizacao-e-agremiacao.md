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

### 2. Afiliação — vínculo fraco, opcional e N:N

- **Opcional:** `Institution` pode existir sem vínculo (`organization_id` nulo ou sem linhas em junção).
- **Fraco e mutável:** pode mudar de organização e pode estar vinculada a **mais de uma** ao mesmo tempo.
- **Modelagem proposta (N:N fraco):** tabela de junção `institution_organizations` em vez de FK único em `institutions`:

```sql
CREATE TABLE platform.institutions (
  id uuid PRIMARY KEY,
  name text NOT NULL,
  type text NOT NULL CHECK (type IN ('CLUB','UNIVERSITY')),
  status text NOT NULL,
  colors text[], -- paleta (ver §5.2)
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE platform.institution_organizations (
  institution_id uuid NOT NULL REFERENCES platform.institutions(id) ON DELETE CASCADE,
  organization_id uuid NOT NULL REFERENCES platform.organizations(id) ON DELETE CASCADE,
  created_at timestamptz NOT NULL DEFAULT now(),
  created_by uuid,
  PRIMARY KEY (institution_id, organization_id)
);
-- Índice inverso para busca por organização
CREATE INDEX idx_institution_orgs_org ON platform.institution_organizations(organization_id);
```

- **Alternativa descartada:** `institutions.organization_id` FK único — limita a 1:1 e não reflete "pode se associar a mais de uma".
- **Auditoria de transferência:** se necessário histórico, adicionar `institution_organization_history (id, institution_id, from_organization_id, to_organization_id, changed_at, changed_by)` ou usar tabela de auditoria genérica; MVP pode ser só `created_at/created_by` na junção + log de app.

**Exemplos:**
- Clube sem vínculo: linhas só em `institutions`, nenhuma em `institution_organizations`.
- Clube em 2 ligas: duas linhas em `institution_organizations` com `organization_id` distintos.
- Troca de liga: `DELETE` + `INSERT` na junção (ou `INSERT`/`DELETE` parcial para múltiplas).

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
- Migration para `institutions` + `institution_organizations` (N:N fraco, opcional); **não** reaproveitar `organizations`.
- `SecurityExpressions`: `ORGANIZATION_WRITE = "hasAuthority('STATUS_ACTIVE') and hasAnyRole('ORGANIZER','ADMIN','ADMIN_LIGA')"` e `INSTITUTION_WRITE = "hasAuthority('STATUS_ACTIVE') and hasAnyRole('ORGANIZER','MANAGER','ADMIN','ADMIN_LIGA')"`.
- Frontend: separar navegação/cards "Organizações" vs "Instituições" (label PT: Agremiações); **Home order:** `Organizações` → `Agremiações` → demais cards; novo card "Agremiações" aponta para módulo `institutions`.

### 5. Melhorias de UX (backlog anotado, sem implementação)

#### 5.1 Dropdown — lista descola ao scrollar
- **Problema:** ao abrir o dropdown e rolar a página, o overlay da lista acompanha o scroll e descola do campo.
- **Diretriz:** usar portal/overlay ancorado ao campo (`OverlayEntry`/`CompositedTransformFollower` ou `DropdownButtonFormField` com menu em overlay nativo); lista deve reposicionar/fechar ao scroll ou resize. Registrar como bug de `KicksterDropdown` em `flag_core`.

#### 5.2 Modal de escolha de cores — fora do DS

**(a) Padrão do Design System (referência Figma)**

- Referência: [Kickster — Popup "Share this Match" — node 34430:8519](https://www.figma.com/design/bXGRAtra3DkMAPGKLLLaCQ/Kickster---Live-Score---News-Sport-Apps-UI-Kits--Community-?node-id=34430-8519&t=4VvSQu6LIHJAB1cW-4) — frame **343px**, padding **24px**, gap **20px**, fundo `color.surface (#FFFFFF)`, **raio 24**, sombra `0 8 32 rgba(18,25,51,0.06)`, header `Body Large Bold` + botão close circular (`surface.muted`, 24px), divider `Grayscale 20 (#ECF1F6)` 1px.
- Tokens a registrar em `design/tokens.md` (se faltarem): `radius.modal = 24`, `elevation.modal = 0 8 32 / 6%`, `modal.padding = 24`, `modal.gap = 20`, `modal.divider = #ECF1F6`, `modal.maxWidth = 343` (mobile) / `480` (web).

**(b) Paleta — de 4 cores fixas para paleta editável**

- Substituir as 4 cores fixas por **seletor editável**: usuário pode adicionar/remover cores (color picker nativo).
- **3 cores principais** escolhidas refletem na identidade (ex: `primary`/`secondary`/`accent` da Institution/Organization) — preview ao vivo no card/header.
- Persistir paleta como `institution.colors: string[]` (hex), com validação e limite (ex: até 6).

## Consequências
- Clareza de domínio e de permissões.
- Migração de dados necessária se houver clubes/universidades já em `organizations`.

### 6. Home — novo card Agremiações

- Novo `KicksterCard` "Agremiações" no `flag_admin_web` Home, refletindo o módulo `institutions`.
- **Ordem dos cards:** `Organizações` (1º) → `Agremiações` (2º) → demais cards (Competições, Times, etc.).

## Não-escopo deste ADR
Implementação, migrations, APIs e telas — aguardar demais pontos do PO.

## Referências
- ADR-003 (club entity hierarchy)
- ADR-008 (PENDING read-only)
- `architecture/roles-and-permissions.md`
