# ADR-001 — Nova Filosofia de Arquitetura e Simplificação

**Status:** Proposto  
**Data:** 2026-09-05  
**Autor:** Tech Lead (Flag Platform)

---

## Contexto

O Flag Platform está em estágio inicial de desenvolvimento (muitas issues em aberto, nenhuma em produção). A arquitetura atual é um **Modular Monolith** com Spring Boot + PostgreSQL + JWT custom. 

Durante análise de mercado comparativa com FlagRoster, Flag50, TeamSnap e GameChanger, identificamos gaps críticos:

1. **Hierarquia confusa**: Time = Clube (mesmo conceito duplo)
2. **Roles limitados**: Apenas ADMIN, ORGANIZER, MESA (3 roles)
3. **Regras manuais**: Tiebreakers, walkover, referee rotation sem automação
4. **Auth não centralizado**: JWT custom separado (difícil de gerenciar)
5. **Apps fragmentados**: Sem integração entre Admin Web, Referee App, Public App

---

## Decisão

Adotar uma **nova filosofia de arquitetura** focada em:

### 1. Simplificação da Hierarquia (5 níveis máximos)

```
1. Organization (Federação/Liga)       [1]:[N]
   └── Club (Clube/Universidade)        [1]:[N]
        └── Team (Time de competição)   [1]:[N]
             └── Roster (Elenco por Season) [1]:[N]
                  └── Athlete (Atleta) [1]:[N]
```

**Elimina a confusão** Time vs Clube. Cada conceito tem sua própria entidade.

### 2. Roles Expandidas (Modelo C - 4 Layers)

```
SUPER_ADMIN (1)     → Acesso total + gestão de orgs
    ↓
ORG_ADMIN (N)       → Gestão de clube/liga específica
    ↓
MANAGER (N)         → Cadastros dentro do clube
    ↓
USER (N)            → Atleta/coach/referee com roles específicos
```

**Expande de 3 para 4+ roles**, permitindo controle de acesso mais granular sem complexidade excessiva.

### 3. Autenticação Centralizada (Firebase Auth + Custom Claims)

- **Migrar JWT custom para Firebase Auth**
- **Usar Custom Claims para roles/permissões**
- **Tokens contêm role + skills** (athlete, coach, referee, manager)
- **Security Rules no Firestore/Realtime Database**

### 4. Apps Especializados (3 + 1 Ferramenta)

| App | Foco | Integração |
|-----|------|------------|
| Admin Web | Gestão de org/clube/time | CRUD completo |
| Referee App | Operação de jogo | Scoring em tempo real |
| Public App | Fan Experience | Standings, highlights |
| Coach Tools | Estratégia | Playbook, stats |

### 5. Regras Automatizadas

- Tiebreakers automáticos
- Walkover tracking
- Referee rotation
- Roster eligibility

### 6. Estratégia de Dados: PostgreSQL Primário + Firestore CQRS Light

```
┌─────────────────────────────────────────────────────────────┐
│                    ARQUITETURA DE DADOS                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐    WRITE PATH    ┌──────────────────┐     │
│  │   Clients   │ ───────────────► │   PostgreSQL     │     │
│  │  (Apps)     │                  │  (Source of Truth)│     │
│  └─────────────┘                  └────────┬─────────┘     │
│                                             │               │
│                              ASYNC SYNC     │               │
│                                             ▼               │
│  ┌─────────────┐    READ PATH     ┌──────────────────┐     │
│  │   Clients   │ ◄──────────────► │    Firestore     │     │
│  │  (Dashboards│                  │  (CQRS Mirror)   │     │
│  │  Leaderboards)                └──────────────────┘     │
│  └─────────────┘                                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Princípios:**

| Camada | Responsabilidade | Tecnologia |
|--------|------------------|------------|
| **Command/Write** | Transações, integridade, validações | PostgreSQL (ACID) |
| **Query/Read** | Dashboards, leaderboards, real-time scores | Firestore (escalável) |
| **Sync** | Event-driven via Cloud Functions | PostgreSQL → Firestore |

**Entidades Espelhadas (Read Model):**

| Entidade | Tabela PostgreSQL | Firestore Collection | Use Case |
|----------|-------------------|----------------------|----------|
| Organization | `organizations` | `organizations` | Dropdowns, listings |
| Competition | `competitions` | `competitions` | Calendar, standings |
| Category | `categories` | `categories` | Division views |
| Team | `teams` | `teams` | Team profiles, rosters |
| Round | `rounds` | `rounds` | Schedule views |
| Game | `games` | `games` | Live scores, details |
| ScoreEvent | `score_events` | `score_events` | Play-by-play |
| Standing | `standings` | `standings` | Leaderboards |
| Athlete | `athletes` | `athletes` | Profiles, stats |
| RosterEntry | `team_roster` | `roster_entries` | Eligibility checks |
| CheckIn | `checkins` | `checkins` | Real-time validation |

**Benefícios:**
- ✅ **Escrita consistente** – PostgreSQL garante ACID para scores, check-ins, limits
- ✅ **Leitura performática** – Firestore escalável para dashboards, analytics
- ✅ **Custo controlado** – Ambos gratuitos/low-cost no tier inicial
- ✅ **Zero refatoração de schema** – Dados de domínio continuam em PostgreSQL

---

## Consequências

### Positivas
- ✅ **Menos confusão**: Hierarquia clara org→clube→time→elenco→atleta
- ✅ **Mais controle**: 4 roles granulares em vez de 3
- ✅ **Auth simplificado**: Firebase Auth com custom claims
- ✅ **Apps integrados**: Todos compartilham o mesmo backend
- ✅ **Escalável**: Pode adicionar novos roles sem refatoração
- ✅ **Leitura otimizada**: Firestore para dashboards e real-time

### Negativas
- ⚠️ **Migração necessária**: Reverter builds anteriores do Firestore
- ⚠️ **Risco de quebra**: Mudança de autenticação pode afetar testes existentes
- ⚠️ **Complexidade inicial**: Implementar Firebase Auth + Custom Claims
- ⚠️ **Tempo**: Requer revisão de issues existentes
- ⚠️ **Dual-write**: Sincronização eventual entre PostgreSQL ↔ Firestore

---

## Alternativas Consideradas

### A. Manter status quo (Modular Monolith + JWT custom)
- Prós: Sem migração, menos risco
- Contras: Hierarquia confusa, roles limitados, auth não escalável

### B. Microsserviços
- Prós: Escalabilidade total
- Contras: Complexidade excessiva, viola ADR-001 original

### C. Firebase somente para Auth (mantém PostgreSQL)
- Prós: Auth centralizado, dados em PostgreSQL
- Contras: Dois sistemas de autenticação, complexidade

### D. Firebase completo (Auth + Firestore)
- Prós: Tudo em um lugar, escalável
- Contras: Requer reverter builds anteriores, lock-in

---

## Decisão Final

**Adotar a Nova Filosofia com:**
- Hierarquia org→clube→time→elenco→atleta
- 4 roles: SUPER_ADMIN, ORG_ADMIN, MANAGER, USER
- Firebase Auth com Custom Claims
- Manter PostgreSQL para dados de domínio (não migrar para Firestore)
- **Firestore como espelho CQRS Light para leitura**
- Apps especializados integrados

---

## Plano de Implementação

### Fase 1: Arquitetura de Roles (2-3 sprints)
1. Expandir `UserRole` enum: SUPER_ADMIN, ORG_ADMIN, MANAGER, USER
2. Implementar Firebase Auth + Custom Claims
3. Mapear permissões por role
4. Criar service de role management

### Fase 2: Arquitetura Flutter (view/model/repository/service) (2 sprints)
**Contexto**: Aplicação `flag_admin_web` será reescrita seguindo o padrão recomendado pelo Flutter para apps maintaináveis e testáveis.

**Premissas adotadas (baseadas em https://docs.flutter.dev/app-architecture/guide e case-study):**

1. **Separação de responsabilidades** — A aplicação será dividida em quatro camadas distintas:
   - **Views**: Widgets que apenas apresentam dados. Nenhuma lógica de negócio. Apenas lógica de layout, animação e condicionais simples baseadas no state do ViewModel.
   - **ViewModels**: Contém a lógica que converte dados brutos em `UI State`. Responsável por recuperar dados de Repositories, transformá-los e expor `commands` (ex: `loadOrganizations`, `deleteOrganization`, `toggleOrganizationSelection`) para os Views reagirem a eventos do usuário.
   - **Repositories**: Fonte de verdade para dados do modelo. Polling de serviços, transformação em domain models, handling de caching, error handling, retry logic. Uma repository por tipo de dados (ex: `OrganizationRepository`).
   - **Services**: Camada mais baixa. Envolve endpoints de API (REST) e exponencia `Future`/`Stream`. Não possui state própria. Um service por fonte de dados (ex: `OrganizationService` wrapping REST API).

2. **One-to-one relationship** — Cada View tem exatamente um ViewModel correspondente. O ViewModel expõe o state necessário para o View renderizar.

3. **Data flow**: View → receives UI State from ViewModel → user interaction → View calls ViewModel command → ViewModel → retrieves/transforms data from Repository → Repository → fetches from Service (REST API) → data reaches View.

4. **Package structure** (padrão Compass app combinado: por tipo + por feature):
   - `lib/ui/<feature_name>/` — por feature: `view_models/<vm>.dart`, `widgets/<screen>.dart`
   - `lib/data/` — por tipo: `repositories/<repo>.dart`, `services/<service>.dart`, `model/<api-model>.dart`

5. **Começando pelo módulo organizations**:
   - Criar `OrganizationService` (serviço REST API)
   - Criar `OrganizationRepository` (source of truth, caching, error handling)
   - Criar `OrganizationViewModel` (transforma dados para UI State, expõe commands)
   - Criar `OrganizationView` (widget que apenas consome o viewModel)

6. **Injeção de dependência** — Usar `provider` ou `get_it` para conectar as camadas, seguindo o padrão do case study.

7. **Testabilidade** — Cada camada pode ser testada isoladamente:
   - Views testadas como widgets unitários
   - ViewModels testados com mock de Repository
   - Repositories testados com mock de Service
   - Services testados com mock de HTTP client

**Critérios de aceitação para o módulo organizations:**
- [ ] `OrganizationViewModel` expõe `Organization[]` state e commands `load`, `delete`, `toggleSelection`
- [ ] `OrganizationRepository` abstrai o serviço e fornece `Organization` domain model
- [ ] `OrganizationService` faz chamadas REST API e retorna `Organization` domain model
- [ ] `OrganizationView` apenas renderiza baseada no state do ViewModel, sem lógica de negócio
- [ ] Testes unitários passando para ViewModel e Repository

#### Estrutura de Pastas Recomendada (flutter app architecture case study)

```
lib/
├── ui/                                          # Organizada por FEATURE
│   ├── core/                                    # Widgets e theme globais compartilhados
│   │   ├── ui/                                  # Shared widgets (buttons, inputs, etc.)
│   │   └── themes/                              # ThemeData da aplicação
│   └── <feature_name>/                         # Pasta por feature (ex: organizations/)
│       ├── view_models/                        # ViewModel classes (1 por feature)
│       │   └── <view_model_class>.dart
│       └── widgets/                            # View widgets (screens + sub-widgets)
│           ├── <feature_name>_screen.dart      # Screen principal
│           └── other_widgets                   # Widgets auxiliares
├── domain/                                      # Modelos de domínio (entities)
│   └── models/                                  # Domain models (ex: Organization, Club, Team, Athlete)
│       └── <model_name>.dart
├── data/                                        # Organizada por TIPO (shared across features)
│   ├── repositories/                           # Repository classes (1 por tipo de dado)
│   │   └── <repository_class>.dart
│   ├── services/                               # Service classes (API clients, lowest layer)
│   │   └── <service_class>.dart
│   └── model/                                  # API models (JSON serialization)
│       └── <api_model_class>.dart
├── config/                                      # Configuração (rotas, inicialização)
├── utils/                                       # Utilitários diversos
├── routing/                                     # Configuração de rotas
├── main_staging.dart                           # Entry point staging
├── main_development.dart                       # Entry point development
└── main.dart                                   # Entry point production

test/                                            # Unit and widget tests
├── data/
├── domain/
├── ui/
└── utils/
testing/                                         # Mocks and utilities for tests
├── fakes/
│   └── models/
```

### Fase 3: ...

### Fase 2: Hierarquia Organizacional (3-4 sprints)
1. Migrar `Team` para `Club → Team`
2. Criar nova tabela `Season`
3. Atualizar `Roster` para ligar a Season/Competition
4. Migração Flyway: versionamento schema

### Fase 3: Core Gameplay (4-5 sprints)
1. Tiebreakers automáticos
2. Walkover tracking
3. Referee rotation
4. Per-play scoring

### Fase 4: Experiência do Usuário (3 sprints)
1. Live scoring + Apple Watch
2. Playbook tools (6v6)
3. Player profiles avançados
4. Live streaming (parceria)

### Fase 5: CQRS Light - Firestore Mirror (2-3 sprints)
1. Configurar Cloud Functions para sincronização
2. Implementar soft deletes em PostgreSQL
3. Migrar dados iniciais (seed) para Firestore
4. Criar endpoints de leitura em Firestore
5. Validar performance e indexação

---

## Critérios de Aceitação

- [ ] Hierarquia org→clube→time→elenco→atleta implementada
- [ ] 4 roles: SUPER_ADMIN, ORG_ADMIN, MANAGER, USER
- [ ] Firebase Auth com Custom Claims
- [ ] Apps integrados via API unificada
- [ ] Tiebreakers automáticos
- [ ] Walkover tracking
- [ ] Referee rotation
- [ ] Per-play scoring
- [ ] **PostgreSQL → Firestore sync ativo**
- [ ] **Dashboards/leaderboards lendo de Firestore**
- [ ] Testes de aceitação passando

---

## Riscos

| Risco | Mitigação |
|-------|-----------|
| Quebra de builds anteriores | Reverter commits de Firestore, manter PostgreSQL |
| Complexidade de migração | Fases graduais, testes em cada sprint |
| Resistência da equipe | Comunicação clara dos benefícios |
| Lock-in Firebase | Usar apenas Auth, manter dados em PostgreSQL |
| **Eventual consistency** | **Aceitável para leitura (dashboards), crítico para escrita (PostgreSQL)** |

---

*Esta ADR substitui a ADR-001 anterior (Filosofia do Projeto).*