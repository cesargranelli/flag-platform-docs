# ADR-001: Refatoração Estrutural — Team, Roster e Season

**Status:** Proposto  
**Data:** 2026-08-31  
**Autor:** Tech Lead (Flag Platform)  

---

## Contexto

A estrutura atual do Flag Platform trata "time" como a **inscrição de um clube em uma competição** (`Team` é uma tabela pivô entre `Organization` e `Competition`). Isso impede que:

- Um clube tenha múltiplos times **independentemente** de competições
- Times tenham elencos **por temporada**
- A separação entre entidade institucional (clube) e representação competitiva (time) seja clara

A plataforma **não está em produção**, portanto a migração pode ser feita com truncamento da base de dados.

## Decisão

Adotar a seguinte hierarquia:

```
Federação / Liga / Associação  [1]:[N]
  └── Clube / Universidade     [1]:[N]
        └── Time               [1]:[N]
              └── Elenco       [1]:[N]  (vinculado a uma Temporada/Competição)
                    └── Atleta [1]:[N]
```

### Conceitos-chave

| Conceito | Definição |
|----------|-----------|
| **Organization** | Entidade institucional (federação, liga, clube, universidade). Permanece como está. |
| **Time** | Sub-entidade de um Clube. Representa o clube nas competições. Existente **antes** de qualquer inscrição. |
| **Competição** | Evento esportivo (campeonato). Tem season (ano/período). Um time é **inscrito** em competições. |
| **Elenco (Roster)** | Composição de atletas de um time **para uma competição/temporada**. Um time pode ter elencos diferentes por competição. |
| **Atleta** | Pessoa atleta. Global na plataforma. Pode estar em múltiplos elencos. |
| **Temporada (Season)** | Período/ano da competição. Campo obrigatório na criação do campeonato. |

---

## Mudanças por Repositório

### 1. Backend (`flag_api`)

#### 1.1 Schema do Banco (PostgreSQL)

**Tabela `team` (NOVA — substitui a atual)**

```sql
CREATE TABLE team (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    name VARCHAR(255) NOT NULL,
    short_name VARCHAR(50),
    sport_name VARCHAR(255),
    logo_url VARCHAR(500),
    status VARCHAR(20) DEFAULT 'ACTIVE',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_team_organization ON team(organization_id);
```

**Tabela `competition_team` (NOVA — substitui a atual `team`)**

```sql
CREATE TABLE competition_team (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    competition_id UUID NOT NULL REFERENCES competition(id),
    team_id UUID NOT NULL REFERENCES team(id),
    division_id UUID REFERENCES division(id),
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(competition_id, team_id)
);

CREATE INDEX idx_competition_team_competition ON competition_team(competition_id);
CREATE INDEX idx_competition_team_team ON competition_team(team_id);
```

**Tabela `roster` (NOVA — separada de `roster_entry`)**

```sql
CREATE TABLE roster (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES team(id),
    competition_id UUID NOT NULL REFERENCES competition(id),
    name VARCHAR(255), -- ex: "Elenco Principal 2026"
    season VARCHAR(50), -- ex: "2026", "2026-Q1"
    status VARCHAR(20) DEFAULT 'ACTIVE',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(team_id, competition_id) -- um elenco por time por competição
);

CREATE INDEX idx_roster_team ON roster(team_id);
CREATE INDEX idx_roster_competition ON roster(competition_id);
```

**Tabela `roster_entry` (ATUALIZADA — referencia `roster` em vez de `team`)**

```sql
ALTER TABLE roster_entry DROP CONSTRAINT IF EXISTS roster_entry_team_id_fkey;
ALTER TABLE roster_entry ADD COLUMN roster_id UUID REFERENCES roster(id);
-- migrar dados existentes: roster_entry → roster → competition_team
ALTER TABLE roster_entry DROP COLUMN team_id;
```

**Tabela `competition` (ATUALIZADA — campo season obrigatório)**

```sql
ALTER TABLE competition ADD COLUMN season VARCHAR(50) NOT NULL DEFAULT '2026';
-- Em produções futuras: remover default e tornar obrigatório na API
```

#### 1.2 Endpoints da API

| Método | Endpoint | Descrição | Status |
|--------|----------|-----------|--------|
| `GET` | `/api/v1/organizations/{orgId}/teams` | Listar times de um clube | **NOVO** |
| `POST` | `/api/v1/organizations/{orgId}/teams` | Criar time dentro de um clube | **NOVO** |
| `GET` | `/api/v1/teams/{teamId}` | Detalhe do time | **NOVO** |
| `PUT` | `/api/v1/teams/{teamId}` | Atualizar time | **NOVO** |
| `DELETE` | `/api/v1/teams/{teamId}` | Remover time | **NOVO** |
| `POST` | `/api/v1/competitions/{compId}/teams/{teamId}` | Inscrever time em competição | **NOVO** (substitui `associateClub`) |
| `DELETE` | `/api/v1/competitions/{compId}/teams/{teamId}` | Desinscrever time | **NOVO** (substitui delete team) |
| `GET` | `/api/v1/competitions/{compId}/teams` | Listar times inscritos | **ATUALIZADO** (era `/clubs`) |
| `GET` | `/api/v1/teams/{teamId}/roster` | Listar elenco do time na competição | **ATUALIZADO** (agora filtra por competition) |
| `POST` | `/api/v1/teams/{teamId}/roster` | Adicionar atleta ao elenco | **ATUALIZADO** (precisa `competitionId`) |
| `DELETE` | `/api/v1/teams/{teamId}/roster/{athleteId}` | Remover atleta do elenco | **ATUALIZADO** |
| `POST` | `/api/v1/teams/{teamId}/roster/batch` | Importação em lote | **ATUALIZADO** |
| `PUT` | `/api/v1/competitions/{compId}` | Atualizar competição | **ATUALIZADO** (campo `season` obrigatório) |
| `POST` | `/api/v1/competitions` | Criar competição | **ATUALIZADO** (campo `season` obrigatório) |

#### 1.3 Regras de Negócio

1. **Time pertence a um clube**: `team.organization_id` obrigatório, deve ser do tipo `club` ou `university`
2. **Inscrição em competição**: Um time só pode ser inscrito em uma competição se tiver pelo menos 1 elenco com atletas
3. **Elenco por competição**: Um time pode ter apenas 1 elenco por competição (pode ter elencos diferentes em competições distintas)
4. **Season na competição**: Campo `season` obrigatório ao criar competição
5. **Atletas globais**: Atletas não são vinculados a organizações — estão em elencos de times

---

### 2. Admin Web (`flag_admin_web`)

#### 2.1 Domain Models

| Modelo | Mudança |
|--------|---------|
| `Team` | **REESCRITO**: remove `competitionId`, adiciona `organizationId` obrigatório |
| `RosterEntry` | **ATUALIZADO**: referencia `rosterId` em vez de `teamId` |
| `Roster` | **NOVO**: `{id, teamId, competitionId, name, season, status}` |
| `Competition` | **ATUALIZADO**: adiciona campo `season` (String) |
| `CompetitionTeam` | **NOVO**: `{id, competitionId, teamId, divisionId}` (join table) |

#### 2.2 API Services

| Service | Mudança |
|---------|---------|
| `TeamApi` | **REESCRITO**: endpoints `/organizations/{orgId}/teams` + `/competitions/{compId}/teams/{teamId}` |
| `RosterApi` | **ATUALIZADO**: endpoints agora usam `rosterId` em vez de `teamId` direto |
| `CompetitionApi` | **ATUALIZADO**: create/update precisam de `season` |

#### 2.3 Screens

| Tela | Mudança | Prioridade |
|------|---------|------------|
| `OrganizationDetailScreen` | **ADICIONAR** seção "Times" — listar/criar times do clube | 🔴 Alta |
| `TeamFormScreen` | **NOVO** — formulário criar/editar time (nome, logo, etc.) | 🔴 Alta |
| `TeamDetailScreen` | **REESCRITO** — mostrar time + elencos por competição | 🔴 Alta |
| `CompetitionCreateScreen` | **ATUALIZADO** — campo `season` obrigatório | 🔴 Alta |
| `CompetitionEditScreen` | **ATUALIZADO** — campo `season` | 🔴 Alta |
| `AssociateClubsScreen` | **REESCRITO** — agora lista **times** disponíveis (não clubes) para inscrever | 🔴 Alta |
| `TeamsScreen` | **REESCRITO** — listar times inscritos na competição (via `competition_team`) | 🔴 Alta |
| `TeamRosterScreen` | **ATUALIZADO** — busca elenco por `competitionId` + `teamId` | 🟡 Média |
| `RostersScreen` | **REESCRITO** — listar elencos por competição, mostrando time → elenco | 🟡 Média |
| `RosterAddAthleteScreen` | **ATUALIZADO** — precisa de `competitionId` para criar elenco | 🟡 Média |

#### 2.4 Navegação (Routes)

| Rota | Mudança |
|------|---------|
| `/organizations/:id` | Adicionar aba/seção "Times" |
| `/teams/new` | **NOVO** — criar time dentro de um clube |
| `/teams/:id` | **REESCRITO** — detalhe do time (elencos por competição) |
| `/teams/:id/edit` | **NOVO** — editar time |
| `/competitions/:id/teams` | **REESCRITO** — listar times inscritos |
| `/competitions/:id/teams/associate` | **REESCRITO** — inscrever time existente |
| `/rosters` | **REESCRITO** — listar elencos por competição |
| `/teams/:id/roster` | **ATUALIZADO** — filtra por competição |

#### 2.5 Providers

| Provider | Mudança |
|----------|---------|
| `teamsProvider(competitionId)` | **MANTIDO** — agora busca via `competition_team` |
| `clubTeamsProvider(orgId)` | **NOVO** — times de um clube |
| `rosterProvider(teamId, competitionId)` | **NOVO** — elenco de um time em uma competição |
| `rosterEntriesProvider(rosterId)` | **NOVO** — atletas de um elenco |
| `effectiveCompetitionProvider` | **ATUALIZADO** —考虑 season |

---

### 3. Public App (`flag_public_app`)

> **Impacto: Médio** — o app público lista times e elencos para o público.

| Componente | Mudança |
|------------|---------|
| Team list | Buscar via `competition_team` em vez de `team` direto |
| Roster view | Buscar via `roster` + `roster_entry` com `competitionId` |
| Team detail | Mostrar elenco da competição selecionada |

**Endpoints consumidos:**
- `GET /api/v1/competitions/{compId}/teams` (substitui listagem atual)
- `GET /api/v1/teams/{teamId}/roster?competitionId={compId}` (novo filtro)

---

### 4. Referee App (`flag_referee_app`)

> **Impacto: Baixo** — o app de árbitros usa `CheckIn` que referencia `teamId`.

| Componente | Mudança |
|------------|---------|
| CheckIn | `teamId` agora referencia a tabela `team` (não mais `competition_team`) |
| Game detail | Lista de times continua usando `competition_team` |

**Endpoints consumidos:**
- `GET /api/v1/games/{gameId}/checkin` — não muda (usa `teamId`)
- `GET /api/v1/competitions/{compId}/teams` — substitui listagem atual

---

## Ordem de Implementação

### Fase 1 — Backend (bloqueante)
1. Criar migração Flyway: truncar tabelas existentes
2. Criar tabela `team` (nova)
3. Criar tabela `competition_team` (nova)
4. Criar tabela `roster` (nova)
5. Atualizar tabela `roster_entry` (referenciar `roster_id`)
6. Atualizar tabela `competition` (campo `season`)
7. Implementar endpoints de Team CRUD
8. Implementar endpoints de CompetitionTeam (inscrever/desinscrever)
9. Atualizar endpoints de Roster
10. Atualizar Competition endpoints (season obrigatório)
11. Documentar Swagger

### Fase 2 — Admin Web
1. Atualizar domain models
2. Atualizar API services
3. Implementar tela de gestão de times no clube
4. Reescrever tela de times da competição
5. Reescrever tela de elencos
6. Atualizar formulário de competição (season)
7. Testes visuais e fluxos

### Fase 3 — Public App + Referee App
1. Atualizar consumo de API
2. Ajustar navegação

---

## Diagrama de Entidades (novo)

```
┌─────────────────────┐
│    Organization     │
│  (clube/univ)       │
└─────────┬───────────┘
          │ 1:N
          ▼
┌─────────────────────┐
│       Team          │
│  (nome, logo, etc)  │
└─────────┬───────────┘
          │ 1:N
          ▼
┌─────────────────────┐     ┌─────────────────────┐
│  CompetitionTeam    │────▶│    Competition      │
│ (inscrição)         │     │ (nome, season, etc) │
└─────────┬───────────┘     └─────────┬───────────┘
          │                           │
          │ 1:N                       │ 1:N
          ▼                           ▼
┌─────────────────────┐     ┌─────────────────────┐
│       Roster        │◀────│    Division/Conf    │
│ (elenco p/ compet)  │     └─────────────────────┘
└─────────┬───────────┘
          │ 1:N
          ▼
┌─────────────────────┐     ┌─────────────────────┐
│    RosterEntry      │────▶│      Athlete        │
│ (posição, número)   │     │ (nome, CPF, etc)    │
└─────────────────────┘     └─────────────────────┘
```

---

## Riscos

| Risco | Mitigação |
|-------|-----------|
| Breaking change na API | Base não está em produção — truncate completo |
| App público e referee precisam atualizar | Comunicar equipe com antecedência |
| Complexidade do elenco por competição | Design simplificado: 1 roster por time/competição |
| Migração de dados | Sem migração — base zerada |

---

## Critérios de Aceitação

- [ ] Clube pode ter N times cadastrados
- [ ] Time pode ser inscrito em N competições
- [ ] Cada inscrição tem seu próprio elenco
- [ ] Competição tem campo `season` obrigatório
- [ ] Elenco é listado por time + competição
- [ ] Atletas são adicionados ao elenco (não ao time diretamente)
- [ ] Todas as telas do admin_web funcionam com a nova estrutura
- [ ] API documentada no Swagger
