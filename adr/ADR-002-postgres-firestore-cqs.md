# ADR-002: Manter PostgreSQL como banco de dados primário + Firestore como espelho CQRS (Light)

## Contexto

A Flag Platform possui um domínio de gestão esportiva (flag football) com:
- Estrutura hierárquica profunda: Organization → Club → Team → Roster → Athlete
- Necessidade de integridade referencial, transações e consultas complexas (leaderboards, estatísticas)
- Volume de dados baixo (10-20 TPS), mas consultas complexas são frequentes
- Existing PostgreSQL schema com 37+ migrations Flyway

## Decisão

**Manter PostgreSQL como banco de dados primário** (sem mudança de backend) e **implementar Firestore como espelho CQRS leve** para consultas de leitura e casos de uso específicos.

### Por quê?

| Critério | PostgreSQL | Firestore |
|----------|------------|------------|
| Integridade referencial | ✅ Garantida | ❌ Manual |
| Transações (scores, limites) | ✅ Suportado | ❌ Não |
| Consultas complexas (join, agregações) | ✅ Nativo | ⚠️ Lenta/Não escalável |
| Auditoria e histórico | ✅ Trácil | ⚠️ Limitado |
| Custo operacional | ✅ Baixo | ✅ Baixo |
| Tempo de implementação | ✅ Já feito | ⚠️ Reestruturação necessária |

### Arquitetura Proposta

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│   Public App │◄────►│  Firestore   │◄────►│  Analytics  │
│  (Web/Flutter)│     │  (Espelho)   │     │  (Leaderboards)│
└─────────────┘     └──────────────┘     └─────────────┘
       │                    │                        │
       ▼                    ▼                        ▼
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│  PostgreSQL │     │  Firestore   │     │  Firestore  │
│  (Primário) │     │  (CQS Mirror)│     │  (CQS Mirror)│
│  - Todos os │     │  - Leituras  │     │  - Dashboards│
│    dados    │     │  - Statistics│     │  - Real-time│
│  - Transações│    │  - Notificações│    │  - Live scores│
└─────────────┘     └──────────────┘     └─────────────┘
```

### Campos de Espelhamento (CQS Light)

| Entidade | Tabela PostgreSQL | Firestore Collection | Observações |
|----------|-------------------|----------------------|-------------|
| Organization | `organizations` | `organizations` | 1:N com Clubs |
| Competition | `competitions` | `competitions` | 1:N com Rounds |
| Category | `categories` | `categories` | 1:N com Rounds |
| Team | `teams` | `teams` | 1:N com RosterEntries |
| Round | `rounds` | `rounds` | 1:N com Games |
| Game | `games` | `games` | 1:N com ScoreEvents |
| ScoreEvent | `score_events` | `score_events` | 1:N com Games (agregado) |
| Standing | `standing` | `standings` | 1:N com Teams/Categories |
| Athlete | `athletes` | `athletes` | 1:N com RosterEntries |
| RosterEntry | `team_roster` | `roster_entries` | 1:N com Games (check-in) |
| CheckIn | `checkins` | `checkins` | 1:N com Games (real-time) |
| User | `users` | `users` | 1:N com Roles |
| PasswordResetToken | `password_reset_tokens` | `password_reset_tokens` | 1:N com Users |

### Benefícios

1. **Preservação do domínio atual** – Nenhuma reestruturação do schema PostgreSQL
2. **Performance de escrita** – Onde importa (inserções de scores, check-ins) permanece em PostgreSQL
3. **Consultas rápidas de leitura** – Dashboards, leaderboards, estatísticas em Firestore
4. **Escalabilidade gradual** – Firestore escala bem para leituras massivas sem impacto na escrita
5. **Custo controlado** – Ambos são gratuitos ou de baixo custo (PostgreSQL gerenciado + Firestore Free Tier)

### Limitações

- **Soft deletes** devem ser implementados em ambas as fontes (PK + `deleted_at`)
- **Transações** que envolvem múltiplas tabelas devem permanecer em PostgreSQL
- **Auditoria** completa (logs de alterações) mantida em PostgreSQL
- **Integração com autenticação** (Firebase Auth) permanece no PostgreSQL (user table)

## Execução

1. **Configurar Cloud Functions** para sincronização bidirecional (PostgreSQL → Firestore)
2. **Implementar soft deletes** em todas as entidades (adicionar `deleted_at` column)
3. **Migrar dados iniciais** (seed) para Firestore
4. **Criar APIs de leitura** em Firestore para dashboards e notificações
5. **Monitorar performance** e ajustar estratégias de indexação

## Próximos Passos

- [ ] Criar Cloud Functions para sincronização CQRS
- [ ] Implementar soft deletes em PostgreSQL
- [ ] Migrar dados iniciais para Firestore
- [ ] Criar endpoints de leitura em Firestore (dashboards, leaderboards)
- [ ] Validar performance de consultas em Firestore

## Referências

- [ADR-001](ADR-001-nova-filosofia-arquitetura.md) – Filosofia inicial
- [Market Analysis](market-analysis-restructured.md) – Comparativo de soluções
- [FlagStats Mapping](flagstats-mapeamento.md) – Mapeamento de métricas

---

**Status**: Approved
**Created**: 2026-09-06
**Version**: 1.0
