# 📊 Análise de Mercado e Recomendações — Flag Platform vs Concorrentes

> **Data:** Setembro 2026  
> **Fontes:** FlagRoster.com, Flag50.com, TeamStats, Ankored, Lumin Sports, pesquisa prévia

---

## 🎯 Visão Geral: Onde o Flag Platform está hoje

### Estrutura Atual do Backend

```
br.com.flagplatform
├── athlete/        → Módulo Atleta
├── checkin/        → Módulo Check-in
├── common/         → Utils compartilhados
├── competition/    → Módulo Campeonato
├── conference/     → Módulo Conferência
├── division/       → Módulo Divisão
├── game/           → Módulo Jogo
├── organization/   → Módulo Organização
├── play/           → Módulo Jogada
├── roster/         → Módulo Elenco
├── round/          → Módulo Rodada
├── security/       → Autenticação JWT
├── standing/       → Módulo Classificação
├── team/           → Módulo Time
├── venue/          → Módulo Campo
├── user/           → Módulo Usuário (ADMIN, ORGANIZER, MESA - 3 roles)
└── FlagPlatformApplication.java
```

### Apps Existentes

| App | Tecnologia | Propósito | Usuários |
|-----|------------|-----------|----------|
| **flag_admin_web** | Flutter Web | Gestão de entidades, cadastros, organização | Organizadores |
| **flag_referee_app** | Flutter Mobile | Operação de jogos, check-in, placar | Mesa/Delegado |
| **flag_public_app** | Flutter Mobile | Acompanhamento público, standings, stats | Atletas/Torcedores |

### Arquitetura

- **Backend:** Java Spring Boot 4.2+, Modular Monolith
- **Banco:** PostgreSQL + Flyway (migrations)
- **Auth:** JWT custom (não Firebase)
- **Frontend:** Flutter 3.x com Riverpod/GoRouter

---

## 🔍 Comparativo com Concorrentes

### 1. FlagRoster — A Plataforma "Todo em Um"

| Aspecto | Flag Platform | FlagRoster | Análise |
|---------|---------------|------------|---------|
| **Foco** | Gestão completa de campeonato | Sazão competitiva completa + regras federação | Flag Platform tem apps separados, FlagRoster integra tudo |
| **Rulesets** | Via módulos | Sim (LFF, NFHS, NFL FLAG) | FlagRoster vence por especialização total |
| **Tiebreakers** | A calcula | Sim (automático) | Diferencial FlagRoster |
| **Walkover/Penalties** | Manuais | Automáticos | Diferencial FlagRoster |
| **Referee Rotation** | Não implementado | Sim | Diferencial FlagRoster |
| **Roster Eligibility** | Manual | Automático | Diferencial FlagRoster |
| **Preço** | Open source? | $499/saison | FlagRoster paga mas completo |
| **Public View** | App separado | Páginas públicas sem login | Similar, mas FlagRoster mais integrado |

**Recomendação:** FlagRoster é a maior ameaça direta para Flag Platform. Se queremos competir, precisamos:
1. Implementar cálculo automático de standings com tiebreakers avançados
2. Adicionar regras específicas (walkover, penalties, referee rotation)
3. Melhorar integração entre apps (hierarquia clara org→clube→time→elenco→atleta)

### 2. Flag50 — Especialista em Game-Day Operations

| Aspecto | Flag Platform | Flag50 | Análise |
|---------|---------------|--------|---------|
| **Live Scoring** | Via Referee App (tap) | Via app árbitro (phone/Apple Watch) + voice scoring | Flag50 mais avançado (Apple Watch + voice) |
| **Per-play Stats** | Parcial | Sim (passer, rusher, receiver) | Flag50 vence |
| **Player Profiles** | Roster básico | Perfil compartilhável com highlights | Flag50 vence |
| **AI Scheduling** | Não | Sim | Diferencial Flag50 |
| **Live Streaming** | Não | Sim (GameChanger integration) | Flag50 vence |
| **Coaching Tools** | Não | Playbook 6v6, scheme finder, AI play builder | Flag50 vence |
| **Age Verification** | Não | Sim (rolling out) | Flag50 tem roadmap |
| **Streaming/Overlays** | Não | Sim | Flag50 tem integração com GameChanger |

**Recomendação:** Flag50 domina o lado "game-day". O Flag Platform deve:
1. Implementar Apple Watch/voice scoring no Referee App
2. Adicionar per-play stat attribution
3. Criar playbook/coach tools dedicados
4. Integrar live streaming simples

### 3. TeamSnap — General Team Admin (Genérico)

| Aspecto | Flag Platform | TeamSnap | Análise |
|---------|---------------|----------|---------|
| **Multi-esporte** | Flag Football apenas | Sim (100+ esportes) | TeamSnap vence por alcance |
| **Schedule** | Custom (divisão/rodadas) | Simples (calendário compartilhado) | Similar |
| **Messages** | Não implementado | Sim (chat por equipe) | TeamSnap vence |
| **Availability** | Não | Sim | TeamSnap vence |
| **Payments** | Não | Sim | TeamSnap vence |
| **Rosters** | Módulo específico | Genérico | Flag Platform vence por especialização |

**Recomendação:** TeamSnap é fraco para flag football específico. Podemos usar ideias de UX (como o sistema de mensagens) mas manter nosso foco especializado.

### 4. GameChanger — Scorekeeping + Fan Streaming

| Aspecto | Flag Platform | GameChanger | Análise |
|---------|---------------|-------------|---------|
| **Fan Experience** | App público simples | Muito avançado (streaming, clips) | GameChanger vence |
| **Live Clips** | Não | Sim (auto-gerados) | Diferencial GameChanger |
| **Box Scores** | Parcial | Sim | GameChanger vence |
| **Parent Access** | Sim (público) | Sim (pago) | Similar |
| **Scorekeeping** | Referee App | App móvel otimizado | Similar qualidade |

**Recomendação:** Para o público, podemos inspirar-nos em GameChanger para:
1. Melhorar design do Public App
2. Adicionar compartilhamento de resultados
3. Integrar clips automáticos (via parceria ou Firebase)

---

## 🎨 Gap Analysis — O que falta implementar?

### Gaps Técnicos (maiores):

| Gap | Concorrente | Impacto | Complexidade |
|-----|-------------|---------|--------------|
| **Tiebreakers automáticos** | FlagRoster | Alto | Média |
| **Walkover/Roster Eligibility** | FlagRoard | Alto | Média |
| **Referee Rotation** | FlagRostar | Médio | Baixa |
| **Per-play stat attribution** | Flag50 | Alto | Baixa |
| **Apple Watch/voice scoring** | Flag50 | Alto | Alta |
| **Playbook tools** | Flag50 | Médio | Alta |
| **Live streaming** | Flag50 | Baixo | Muito Alta |

### Gaps Arquiteturais:

| Gap | Problema | Solução |
|-----|----------|---------|
| **Hierarquia confusa** | Time vs Clube | Implementar nova hierarquia org→clube→time→elenco→atleta |
| **Roles limitados** | Apenas ADMIN/ORGANIZER/MESA | Expandir para SUPER_ADMIN, ADMIN, MANAGER, USER |
| **Auth não centralizado** | JWT custom separado | Migrar para Firebase Auth com custom claims |
| **Dados fragmentados** | App separado do backend | Usar Firebase Functions para sync |

---

## 🛠️ Recomendações de Reestruturação

### 📐 Arquitetura Recomendada

```
                    ┌─────────────────────────────────────┐
                    │       FLAG PLATFORM 2.0             │
                    │   (Arquitetura orientada a serviços)│
                    └─────────────────┬───────────────────┘
                                      │
              ┌───────────────────────┼───────────────────────┐
              │                       │                       │
      Firebase Auth          Spring Boot Modular        Flutter Apps
    (Custom Claims)          (Domínios por módulo)     (3 apps + coach tools)
              │                       │                       │
              ├───────────────────────┤                       │
              │                       ├───────────────────────┤
              ▼                       ▼                       ▼
    ┌──────────────────┐    ┌─────────────────┐    ┌──────────────────┐
    │ Custom Claims   │    │ athletics       │    │ admin_web        │
    │ (roles, skills) │◄──►│ - organization  │◄──►│ - org mgmt        │
    └──────────────────┘    │ - competition │    │ - team mgmt       │
              │             │ - category    │    │ - season mgmt     │
              │             │ - venue       │    │ - user mgmt       │
              │             │ - team        │    │                  │
              │             │ - roster      │    │ referee_app      │
              │             │ - athlete     │◄──►│ - game ops        │
              │             │ - game        │    │ - check-in        │
              │             │ - standing    │    │ - scoring         │
              │             │ - checkin     │    │                  │
              │             │               │    │ public_app       │
              │             │ play          │◄──►│ - standings       │
              └─────────────┴───────────────┴────│ - games         │
                                                │ - player view   │
                                                │                  │
                                                │ coach_tools      │
                                                │ - playbook        │
                                                │ - stats view     │
                                                └──────────────────┘
```

### 📋 Roadmap de Implementação (Modelo C - Simplified)

#### Fase 1: Arquitetura de Roles (2-3 sprints)
- ✅ Expandir `UserRole` enum: adicionar SUPER_ADMIN, MANAGER, USER
- ✅ Implementar custom claims no Firebase Auth
- ✅ Mapear permissões por role
- ✅ Criar service de role management

#### Fase 2: Hierarquia Organizacional (3-4 sprints)
- ✅ Migrar `Team` para `Club → Team`
- ✅ Criar nova tabela `Season` 
- ✅ Atualizar `Roster` para ligar a Season/Competition
- ✅ Migração Flyway: versionamento schema

#### Fase 3: Core Gameplay (4-5 sprints)
- ✅ Tiebreakers automáticos
- ✅ Walkover tracking
- ✅ Referee rotation
- ✅ Per-play scoring

#### Fase 4: Experiência do Usuário (3 sprints)
- ✅ Live scoring + Apple Watch
- ✅ Playbook tools (6v6)
- ✅ Player profiles avançados
- ✅ Live streaming (parceria)

---

## 🎯 Prioridades para Simplificação (Modelo C)

### 1. Simplificar Roles → **Modelo C: 4 Layers de Acesso**

```
SUPER_ADMIN (1)     → Acesso total + gestão de orgs
    ↓
ORG_ADMIN (N)       → Gestão de clube/liga específica
    ↓
MANAGER (N)         → Cadastros dentro do clube
    ↓
USER (N)            → Atleta/coach/referee com roles específicos
```

### 2. Hierarquia Clara → **5 Níveis Máximos**

1. **Organization** (Federação/Liga)
2. **Club** (Clube/Universidade)  
3. **Team** (Time de competição)
4. **Roster** (Elenco por Season)
5. **Athlete** (Atleta)

### 3. Funções Separadas → **3 Apps + 1 Ferramenta**

| App | Foco | Integração |
|-----|------|------------|
| Admin Web | Gestão de org/clube/time | CRUD completo |
| Referee App | Operação de jogo | Scoring em tempo real |
| Public App | Fan Experience | Standings, highlights |
| Coach Tools | Estratégia | Playbook, stats |

---

## 💰 Estratégia de Monetização

| Recurso | Modelo | Preço Sugerido |
|---------|--------|----------------|
| **Standar** | Free + 3.5% transaction | $0 setup |
| **Pro** | $99/mês | Todos os recursos básicos |
| **Elite** | $299/mês | + Playoff, AI Scheduling, Live Video |
| **Coach Tools** | Gratuito | Playbook 6v6 |
| **Verificação de Idade** | $2/player | Compliance |

---

## ✅ Próximos Passos Imediatos

1. **Criar ADR-001 novo** com a nova filosofia: "Simplicidade focada em flag football, integração entre apps, regras automatizadas"
2. **Criar Issue #1**: "Implementar hierarquia org→clube→time→elenco→atleta" 
3. **Criar Issue #2**: "Expandir UserRole enum para SUPER_ADMIN, MANAGER, USER"
4. **Delegar para backend**: Auth service + custom claims
5. **Delegar para frontend**: Role-based navigation

---

## 📊 Resumo Executivo

**O Flag Platform tem uma vantagem única:** Apps nativos dedicados para cada persona (organizador, árbitro, atleta/torcedor) que se comunicam via API unificada.

**Mas precisa evoluir:**
1. **Roles → Firebase Auth + Custom Claims** (simplificar auth)
2. **Hierarquia → org→clube→time→elenco→atleta** (eliminar confusão)
3. **Regras automáticas → tiebreakers, walkover, roster eligibility** (competir com FlagRoster)
4. **Play-by-play → per-play scoring + Apple Watch** (competir com Flag50)
5. **Fan experience → highlights + streaming** (competir com GameChanger)

**Com essas mudanças, o Flag Platform pode ser a única plataforma que:**
- Tem apps nativos otimizados (não web adaptado)
- Oferece integração completa (gestão + jogo + fan)
- Implementa regras Flag Football específicas
- Mantém preço acessível (vs $499 do FlagRoster)

---

*Documento preparado para revisão e aprovação via ADR process.