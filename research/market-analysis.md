# Flag Platform — Análise de Mercado e Comparativo (Set/2026)

> Documento gerado a partir de pesquisas web sobre concorrentes e oportunidades de simplificação/otimização do Flag Platform.

## 📋 Visão Geral

A Flag Platform já possui **fundação sólida** em gestão de campeonato (org → competition → categories → teams → games → standings), diferencial frente a concorrentes que focam em stats ou league management isoladamente.

As maiores oportunidades são:
1. Preencher gaps de **statistics atleta** e **scouting guiado** (requests #1 da pesquisa)
2. Não tentar copiar tudo dos concorrentes — focar no que as 3 apps já fazem bem (public/referee/admin)
3. **Simplificar** — modular monolith atual é vantagem, não migre prematuramente
4. **Etapas graduais** — releases 0.2, 0.3, 0.4 ao invés de resolver tudo de uma vez

---

## 🔍 Pesquisa de Mercado (Fontes: web search deep, setembro 2026)

### Principais Concorrentes Identificados

| Concorrente | Focus | Principais Features | Preço/Modalidade |
|-------------|-------|---------------------|------------------|
| **Flag50** | Ligas/torneios flag football | Registro com pagamentos por jogador, agendamento AI, live scoring por play, perfis de jogadores compartilhados | Platform/SaaS |
| **FlagStat** | Stat tracking & league management | Live scoring, NFHS/NFL FLAG/NIRSA rulesets, tournament brackets, film scoring, compliance dashboards, MaxPreps partner | Web + iPhone app |
| **StatHawk** | Stat tracking focused | Free iPhone app, MaxPreps partner, AI season summaries (Pro), points per drive (free) | Annual subscription |
| **GameChanger** | Multi-sport originally baseball | Scorekeeping, box scores, auto-video clips (Live), fan feed | Free for coaches, $9.99/mês pais |
| **Breakaway Data** | Web platform flag football | Live scoring, player stats, league scheduling, standings, tournament brackets, online registration, parent/player portal | SaaS |
| **SquadDeck** | All-in-one club management | Free club website, event/league management, real-time comm (SMS/email), team management, attendance tracking | Free small leagues; premium $9.10/mês |
| **LeagueApps** | All-in-one sports org | Registration management, payment processing, team management, communication tools, scheduling, reporting/analytics | Tiered pricing |
| **SportLoMo** | Competition management | Fixtures/scheduling, referee assignment, realtime scores/tables/stats, team/individual registration, payments/fees, communication, dashboards | Custom pricing |
| **The Flag Football App** | Social networking | 409 competitors indicated market need | — |
| **QwikCut** | Video analysis | Playbook manager, player grading, video storage/sharing | Demo-based |

---

## 📊 Comparativo Direto: Flag Platform vs Mercado

| Feature | Flag Platform | Principais Concorrentes | Gap/Oportunidade |
|---------|---------------|------------------------|------------------|
| **Gestão completa de campeonato** | ✅ Org → Competition → Category → Venue → Team → Round → Game → Standing | ✅ Flag50, ✅ LeagueApps, ✅ SportLoMo | FlagPlatform já tem isso coberto |
| **Rulesets específicos (NFHS/NFL FLAG/NIRSA)** | ⚠️ Implementado via módulos | ✅ FlagStat, ✅ StatHawk | FlagPlatform pode expandir |
| **Live scoring play-by-play** | ✅ Via Referee App | ✅ FlagStat, ✅ Breakaway Data | Implementar mais visualização |
| **Stat tracking por atleta** | ⚠️ Roster básico | ✅ FlagStat, ✅ StatHawk, ✅ Breakaway | **Oportunidade key** - adicionar histórico de atleta |
| **Escouting guiado de jogadas** | ❌ Gap identificado | ❌ Todos os concorrentes | **Oportunidade key** - novo módulo |
| **Integração MaxPreps/exportação** | ❌ Não implementado | ✅ FlagStat (partner desde mar/2026), ✅ StatHawk | **Oportunidade** - parceria ou implementação |
| **Perfis de atleta com histórico** | ⚠️ Apenas roster atual | ⚠️ Limitado em todos | **Oportunidade** - módulo de histórico |
| **Treinos como categoria de coleta** | ❌ Não implementado | ❌ Gap geral | **Oportunidade** - expandir domínio |
| **Comunidade/social** | ⚠️ Public app sem login | ✅ The Flag Football App (409 competitors), ✅ SquadDeck | **Oportunidade** - features sociais leves |
| **Video highlights/film** | ❌ Não implementado | ✅ QwikCut, ✅ GameChanger (video clips) | **Oportunidade** - parceria ou módulo |

---

## 🎯 Principais Oportunidades de Simplificação & Otimização

### 1. Focar no "Core Value" — Gestão de Campeonato
- FlagPlatform já executa bem: org → competition → categories → teams → games → standings
- **Simplificar**: reduzir complexidade desnecessária nos módulos laterais
- **Priorizar**: experiência integrada 3 apps (public/referee/admin)

### 2. Módulo de Estatísticas Atletas (Gap Crítico)
- Concorrentes: FlagStat, StatHawk focam nisso
- **O que agregar**: integrar com a estrutura existente (athlete → roster → competition)
- **Etapas**:
  - Fase 1: stats básicas por jogo (check-in já existente)
  - Fase 2: histórico por campeonato/amistoso/treino
  - Fase 3: comparação e ranking

### 3. Scouting/Guiado de Jogadas (Gap Identificado)
- Pesquisa #1 dos usuários (do research/flagstats-mapeamento.md)
- **Simplificar**: não tentar copiar FlagStat totalmente
- **Abordagem own**: fluxo simplificado de "play registration" vs complexo scouting
- **Etapas**:
  - Wizard de jogadas por pergunta (como identificado no research)
  - Tipos básicos: corrida, passe, penalty, touchdown
  - Dados mínimos: yards, flags pulled, resultado

### 4. Exportação/Integração com MaxPreps/Entidades
- Concorrentes têm isso como differentiator key
- **Simplificar**: implementar o mínimo viable (formato CSV básico)
- **Prioridade**: dependente do público-alvo (schools/colleges vs ligas comunitárias)

### 5. Features Sociais Leves
- The Flag Football App tem 409 competitors, indicating market need
- **O que FlagPlatform pode fazer diferente**: integrar com as 3 apps já existentes
- **Não tentar ser rede social completa** — features pontuais de compartilhamento

---

## 📈 Roadmap de Simplificação (Recomendado)

| Release | Foco | Status |
|---------|------|--------|
| **0.1 — Championship Foundation** | Org, Competition, Category, Venue, Team, Round, Game, Standing | ✅ Já existente |
| **0.2 — Statistics Basic** | Stat tracking básico por atleta por jogo; check-in validation | 🔜 Planejado |
| **0.3 — Scout Simplificado** | Fluxo de registro de jogadas por pergunta; tipos simplificados | 🔜 Planejado |
| **0.4 — Historian Athlete** | Histórico por atleta por campeonato; comparação simples; exportação CSV básica | 🔜 Planejado |
| **1.0 — Market Fit** | Todos os gaps cobertos de forma simplificada; APIs documentadas; deploy estável | 📅 Futuro |

---

## 💡 Recomendações de Arquitetura

### Manter o Modular Monolith (ADR-003)
- **Vantagem**: simplicidade vs microsserviços
- **Manter boundaries claros** entre módulos (domain-driven design)
- **Não migrar para Firebase agora** — complexidade desnecessária

### Por que NÃO migrar para Firebase agora?

1. **Complexidade desnecessária**: projeto já complexo, Firebase adiciona camada nova
2. **Lock-in**: dados Firebase difíceis migrar depois
3. **Features atuais já atendem**: PostgreSQL + Flyway já funcional
4. **Roadmap phase 0.1-0.4 já definidas** — focar nisso primeiro

### Migração gradual recomendada (se decidir depois)

1. Phase 1: Extração de domínios críticos (athlete, competition) para serviço separado
2. Phase 2: API layer que suporta ambos (PostgreSQL + Firebase)
3. Phase 3: Gradual migration de usuários/novos projetos

---

## 🏁 Próximos Passos Sugeridos

1. **Criar issue formalizada** sobre análise de mercado e oportunidades
2. **Planejar Release 0.2** — statistics básicas (foco no gap atleta)
3. **Planejar Release 0.3** — scout simplificado (fluxo por pergunta)
4. **Definir critérios de aceitação** para cada release
5. **Manter foco** nas 3 apps (public/referee/admin) em vez de tentar ser tudo para todos

---

## 📝 Conclusão

O Flag Platform tem **fundação sólida** em gestão de campeonato (diferencial vs concorrentes que focam em stats ou league management isoladamente). As maiores oportunidades são:

1. **Preencher gaps de statistics atleta** e **scouting guiado** — são os #1 requests da pesquisa
2. **Não tentar copiar tudo** dos concorrentes — focar no que as 3 apps já fazem bem
3. **Simplificar** — o modular monolith atual é vantagem, não migre prematuramente
4. **Etapas graduais** — releases 0.2, 0.3, 0.4 ao invés de tentar resolver tudo de uma vez

---
*Documento gerado: 2026-09-05*
*Baseado em: web search deep, análise de research/flagstats-mapeamento.md, ADRs do projeto*