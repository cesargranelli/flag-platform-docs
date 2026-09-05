# ADR-004: Migração de Autenticação para Firebase Auth + Custom Claims

## Status

Proposto — 2026-09-06

## Contexto

A Flag Platform atualmente utiliza um sistema de autenticação JWT custom (HS256) implementado no Spring Boot:
- Tokens gerados no backend com `JwtTokenProvider`
- Roles armazenados no PostgreSQL (`users.role`)
- Validação via `JwtAuthenticationFilter`

**Problemas identificados:**
1. Autenticação fragmentada entre apps (Admin Web, Referee App, Public App)
2. Sem suporte a OAuth providers (Google, Apple, Facebook)
3. Sem MFA nativo
4. Gerenciamento de sessões分散ado
5. Código custom de autenticação requer manutenção

## Decisão

Migrar para **Firebase Auth** como identity provider, mantendo o **PostgreSQL como source of truth** para roles e permissões.

### Arquitetura Proposta

```
┌──────────────────────────────────────────────────────────────────┐
│                         CLIENTS                                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐         │
│  │ Admin Web   │  │ Referee App│  │ Public App      │         │
│  │ (Flutter)   │  │ (Flutter)  │  │ (Flutter)       │         │
│  └──────┬──────┘  └──────┬──────┘  └────────┬────────┘         │
│         │                │                    │                   │
│         └────────────────┼────────────────────┘                   │
│                          ▼                                        │
│                   Firebase Auth SDK                               │
│                   (ID Token JWT)                                 │
└─────────────────────────────┬────────────────────────────────────┘
                              │ Bearer Token
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│                         BACKEND                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ FirebaseIdTokenFilter                                         │ │
│  │ 1. Verify ID Token (Firebase Admin SDK)                       │ │
│  │ 2. Lookup user by firebase_uid in PostgreSQL                   │ │
│  │ 3. Load roles/permissions from PostgreSQL                    │ │
│  │ 4. Create Spring Security Authentication                     │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                              │                                    │
│                              ▼                                    │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ PostgreSQL (Source of Truth)                                 │ │
│  │ users.firebase_uid | users.role | users.organization_id       │ │
│  └─────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

### Custom Claims (Firebase)

```json
{
  "sub": "firebase-uid-abc123",
  "role": "ORG_ADMIN",
  "organization_id": "uuid-da-org",
  "club_id": "uuid-do-clube",
  "skills": ["athlete", "coach"],
  "email": "user@flagplatform.com",
  "email_verified": true
}
```

### Hierarquia de Roles

| Role | Acesso | Custom Claims |
|------|--------|--------------|
| SUPER_ADMIN | Total, todas orgs | `role: "SUPER_ADMIN"` |
| ORG_ADMIN | Organização específica | `role: "ORG_ADMIN"`, `organization_id: uuid` |
| MANAGER | Clube específico | `role: "MANAGER"`, `club_id: uuid` |
| USER | Básico | `role: "USER"`, `skills: [...]` |

## Alternativas Consideradas

### A. Manter JWT Custom apenas
- **Prós:** Sem migração, controle total do código
- **Contras:** Autenticação fragmentada, sem OAuth/MFA, manutenção custom

### B. Firebase completo (roles no Firebase)
- **Prós:** Tudo no Firebase, menos queries
- **Contras:** Lock-in, perder relacionamentos SQL, сложность миграции

### C. Auth0/Okta
- **Prós:** Enterprise-ready, múltiplos providers
- **Contras:** Custo adicional ($), complexidade

### D. AWS Cognito
- **Prós:** Integração AWS, Lambda triggers
- **Contras:** Menos suportado em Flutter, lock-in AWS

## Consequências

### Positivas
- ✅ **Autenticação unificada** entre todos os apps
- ✅ **OAuth providers** (Google, Apple, Facebook)
- ✅ **MFA nativo** disponível
- ✅ **Sessão gerenciada** pelo Firebase SDK
- ✅ **Menor código custom** de autenticação
- ✅ **Telefone/Email verification** built-in

### Negativas
- ⚠️ **Dependência Firebase** (mas mantemos PostgreSQL como backup)
- ⚠️ **Custom Claims tem limite** de 1000 bytes
- ⚠️ **Migração de usuários** existentes requer coordenação
- ⚠️ **Latência adicional** na validação de token

## Implementação

Ver documento detalhado: [plano-migracao-firebase-auth.md](plano-migracao-firebase-auth.md)

### Fases:
1. **Configuração Firebase** (1 sprint)
2. **Integração Backend** (1 sprint)
3. **Apps Flutter** (2 sprints)
4. **Migração Usuários** (1 sprint)
5. **Decomissionar JWT** (1 sprint)

### Timeline: ~12 semanas (6 sprints)

## Riscos

| Risco | Mitigação |
|-------|-----------|
| Perda de sessão | Blue-green deploy, coexistência JWT/Firebase |
| Custom Claims não propagam | Refresh token, cache local |
| Lock-in Firebase | PostgreSQL como source of truth, abstração |
| Limite 1000 bytes claims | Claims minimalistas, buscar detalhes no PostgreSQL |

## Critérios de Aceitação

- [ ] Todos os apps usam Firebase Auth SDK
- [ ] Backend valida Firebase ID Token + PostgreSQL roles
- [ ] Custom Claims propagam corretamente
- [ ] Usuários existentes migrados
- [ ] JWT custom removido
- [ ] Testes E2E passando
- [ ] Zero downtime na migração

---

**Autor:** Tech Lead (Flag Platform)
