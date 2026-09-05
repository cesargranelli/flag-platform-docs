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

---

## Consequências

### Positivas
- ✅ **Menos confusão**: Hierarquia clara org→clube→time→elenco→atleta
- ✅ **Mais controle**: 4 roles granulares em vez de 3
- ✅ **Auth simplificado**: Firebase Auth com custom claims
- ✅ **Apps integrados**: Todos compartilham o mesmo backend
- ✅ **Escalável**: Pode adicionar novos roles sem refatoração

### Negativas
- ⚠️ **Migração necessária**: Reverter builds anteriores do Firestore
- ⚠️ **Risco de quebra**: Mudança de autenticação pode afetar testes existentes
- ⚠️ **Complexidade inicial**: Implementar Firebase Auth + Custom Claims
- ⚠️ **Tempo**: Requer revisão de issues existentes

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
- Apps especializados integrados

---

## Plano de Implementação

### Fase 1: Arquitetura de Roles (2-3 sprints)
1. Expandir `UserRole` enum: SUPER_ADMIN, ORG_ADMIN, MANAGER, USER
2. Implementar Firebase Auth + Custom Claims
3. Mapear permissões por role
4. Criar service de role management

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
- [ ] Testes de aceitação passando

---

## Riscos

| Risco | Mitigação |
|-------|-----------|
| Quebra de builds anteriores | Reverter commits de Firestore, manter PostgreSQL |
| Complexidade de migração | Fases graduais, testes em cada sprint |
| Resistência da equipe | Comunicação clara dos benefícios |
| Lock-in Firebase | Usar apenas Auth, manter dados em PostgreSQL |

---

*Esta ADR substitui a ADR-001 anterior (Filosofia do Projeto).*