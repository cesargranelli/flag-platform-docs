# Plano de Migração: JWT Custom → Firebase Auth + Custom Claims

## Visão Geral

Migrar o sistema de autenticação da Flag Platform do JWT custom (HS256) para Firebase Auth com Custom Claims, mantendo o PostgreSQL como source of truth para dados de usuário e roles.

## Arquitetura Atual vs Nova

### Atual (JWT Custom)
```
┌──────────┐     ┌─────────────┐     ┌─────────────────┐
│  Client  │────►│  Backend    │────►│  PostgreSQL    │
│          │◄────│  (Spring)   │◄────│  (users table) │
└──────────┘     │  JWT HS256   │     └─────────────────┘
                 └─────────────┘
```

### Nova (Firebase Auth + JWT custom como fallback)
```
┌──────────┐     ┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│  Client  │────►│  Firebase  │────►│  Backend     │────►│  PostgreSQL      │
│          │     │  Auth       │     │  (verify ID  │     │  (users table,  │
│          │◄────│  (ID Token) │     │   token)      │     │   roles)        │
└──────────┘     └─────────────┘     └──────────────┘     └─────────────────┘
```

## Estrutura de Custom Claims

### Hierarquia de Roles (Custom Claims)
```json
{
  "sub": "firebase-uid-xxx",
  "role": "ORG_ADMIN",        // SUPER_ADMIN | ORG_ADMIN | MANAGER | USER
  "organization_id": "uuid",  // ID da organização vinculada (null para SUPER_ADMIN)
  "club_id": "uuid",         // ID do clube vinculado (null para não-Managers)
  "skills": ["athlete"],      // athlete | coach | referee | manager
  "email": "user@email.com",
  "email_verified": true
}
```

### Regras de Precedência
1. `SUPER_ADMIN` tem acesso total (ignore other claims)
2. `ORG_ADMIN` tem acesso à organização especificada
3. `MANAGER` tem acesso ao clube especificado
4. `USER` acesso básico (pode ter skills)

## Fases de Migração

### Fase 1: Configuração Firebase (1 sprint)
**Objetivo:** Configurar Firebase Admin SDK no backend

#### Tasks:
- [ ] Criar projeto Firebase no console
- [ ] Configurar Firebase Admin SDK no `flag_backend`
- [ ] Implementar `FirebaseTokenProvider` (interface existente)
- [ ] Adicionar variáveis de ambiente (FIREBASE_PROJECT_ID, FIREBASE_CREDENTIALS_PATH)
- [ ] Criar serviço `FirebaseAuthService`

#### Artefatos:
```java
// src/main/java/br/com/flagplatform/user/service/FirebaseTokenProvider.java
@Service
public class FirebaseTokenProvider implements TokenProvider {
    
    private final FirebaseAuth firebaseAuth;
    
    @Override
    public String generateToken(String subject) {
        // Para Firebase, não geramos token aqui
        // O token é gerado no cliente via Firebase SDK
        throw new UnsupportedOperationException(
            "Token generation moved to Firebase Auth SDK on client");
    }
    
    public Map<String, Object> generateCustomClaims(UserEntity user) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("role", user.getRole().name());
        claims.put("email", user.getEmail());
        claims.put("organization_id", user.getOrganizationId());
        claims.put("club_id", user.getClubId());
        claims.put("skills", user.getSkills());
        return claims;
    }
    
    public void setCustomClaims(String firebaseUid, Map<String, Object> claims) {
        firebaseAuth.setCustomUserClaims(firebaseUid, claims);
    }
}
```

#### Configuração Spring:
```yaml
# application.yml
firebase:
  project-id: ${FIREBASE_PROJECT_ID:flag-platform}
  credentials-path: ${FIREBASE_CREDENTIALS_PATH:classpath:firebase-credentials.json}
```

---

### Fase 2: Integração Firebase + Backend (1 sprint)
**Objetivo:** Backend aceita tokens Firebase e valida against PostgreSQL

#### Tasks:
- [ ] Implementar `FirebaseIdTokenFilter`
- [ ] Atualizar `SecurityConfig` para aceitar Firebase tokens
- [ ] Criar endpoint para sincronizar user data
- [ ] Implementar mapping Firebase UID ↔ PostgreSQL user

#### Filtro de Autenticação:
```java
@Component
public class FirebaseIdTokenFilter extends OncePerRequestFilter {
    
    private final FirebaseAuth firebaseAuth;
    private final UserRepository userRepository;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                  HttpServletResponse response,
                                  FilterChain chain) throws ServletException, IOException {
        
        String authHeader = request.getHeader("Authorization");
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            String idToken = authHeader.substring(7);
            
            try {
                FirebaseToken decoded = firebaseAuth.verifyIdToken(idToken);
                String firebaseUid = decoded.getUid();
                
                // Buscar usuário no PostgreSQL via Firebase UID
                UserEntity user = userRepository.findByFirebaseUid(firebaseUid)
                    .orElseThrow(() -> new UserNotFoundException(
                        "User not found for Firebase UID: " + firebaseUid));
                
                // Validar role/status no PostgreSQL
                UserPrincipal principal = new UserPrincipal(user, decoded.getClaims());
                SecurityContextHolder.getContext().setAuthentication(
                    new UsernamePasswordAuthenticationToken(
                        principal, null, principal.getAuthorities()));
                        
            } catch (FirebaseAuthException e) {
                response.sendError(HttpServletResponse.SC_UNAUTHORIZED);
                return;
            }
        }
        
        chain.doFilter(request, response);
    }
}
```

---

### Fase 3: Atualização dos Apps Flutter (2 sprints)
**Objetivo:** Apps usam Firebase Auth SDK para login

#### Tasks:
- [ ] Adicionar `firebase_core` e `firebase_auth` aos apps
- [ ] Implementar `FirebaseAuthService` em cada app
- [ ] Atualizar fluxos de login/signup
- [ ] Implementar persistence de sessão
- [ ] Atualizar interceptors para usar ID tokens

#### Estrutura de Código (Flutter):
```dart
// lib/src/auth/firebase_auth_service.dart
class FirebaseAuthService {
  final FirebaseAuth _auth = FirebaseAuth.instance;
  
  Future<UserCredential> signInWithEmailPassword({
    required String email,
    required String password,
  }) async {
    return await _auth.signInWithEmailAndPassword(
      email: email,
      password: password,
    );
  }
  
  Future<void> updateCustomClaims({
    required String role,
    String? organizationId,
    String? clubId,
    List<String>? skills,
  }) async {
    // Chamada para backend que atualiza via Admin SDK
    // Backend deve chamar firebaseAuth.setCustomUserClaims()
  }
  
  String? get idToken => _auth.currentUser?.getIdToken();
}
```

---

### Fase 4: Migração de Usuários Existentes (1 sprint)
**Objetivo:** Migrar usuários do PostgreSQL para Firebase Auth

#### Tasks:
- [ ] Criar script de migração
- [ ] Vincular usuários existentes via `firebase_uid` column
- [ ] Atualizar senhas para Firebase (hash compatibility)
- [ ] Backfill Custom Claims

#### Migração de Dados:
```sql
-- Adicionar coluna firebase_uid
ALTER TABLE users ADD COLUMN firebase_uid VARCHAR(28) UNIQUE;

-- Para cada usuário existente, criar conta Firebase
-- e atualizar firebase_uid (via Admin SDK)
```

#### Script Python (pseudo-código):
```python
for user in users_without_firebase_uid:
    # 1. Criar usuário no Firebase
    firebase_user = auth.create_user(
        email=user.email,
        password=user.password,  # Requer reconfigurar senha
        uid=f"existing_{user.id}"
    )
    
    # 2. Atualizar PostgreSQL
    db.execute(
        "UPDATE users SET firebase_uid = ? WHERE id = ?",
        firebase_user.uid, user.id
    )
    
    # 3. Setar Custom Claims
    auth.set_custom_user_claims(firebase_user.uid, {
        "role": user.role,
        "organization_id": str(user.organization_id),
        "skills": user.skills or []
    })
```

---

### Fase 5: Decomissionar JWT Custom (1 sprint)
**Objetivo:** Remover código JWT custom

#### Tasks:
- [ ] Remover `JwtAuthenticationFilter`
- [ ] Remover `JwtTokenProvider`
- [ ] Remover dependências JWT (jjwt)
- [ ] Limpar configs de JWT
- [ ] Atualizar Swagger/OpenAPI docs

---

## Matriz de Riscos

| Risco | Probabilidade | Impacto | Mitigação |
|-------|-------------|---------|-----------|
| Perda de sessão durante migração | Média | Alto | Blue-green deploy, período de coexistência |
| Custom Claims não propagam imediatamente | Alta | Médio | Cache local, refresh token |
| Usuários sem e-mail verificado | Média | Médio | Forçar verificação antes de operations |
| Limites de Custom Claims (1000 bytes) | Baixa | Médio | Manter claims minimalistas |
| Lock-in Firebase | Baixa | Alto | Manter PostgreSQL como source of truth |

## Custo

| Recurso | Plano | Custo |
|---------|-------|-------|
| Firebase Auth | Spark (Free) | $0 |
| Cloud Functions (se usado) | Spark (Free) | $0 (até 125K invocations/mês) |
| Firebase Admin SDK | N/A | $0 (server-side) |

**Limites free tier:**
- 10k usuários mensais para SMS (se usado)
- 500k usuários para email/password
- Custom Claims: max 1000 bytes por usuário

## Alternativas Consideradas

### 1. Manter JWT Custom apenas
- **Prós:** Sem mudança, controle total
- **Contras:** Autenticação fragmentada, sem benefícios Firebase (MFA, OAuth providers)

### 2. Firebase completo (sem PostgreSQL user)
- **Prós:** Tudo no Firebase
- **Contras:** Lock-in, perder benefícios SQL (relacionamentos, queries complexas)

### 3. Auth0/Okta
- **Prós:** Enterprise-ready
- **Contras:** Custo, complexidade adicional

## Decisão Final

Adotar **Firebase Auth + Custom Claims** com:
1. Firebase como identity provider
2. PostgreSQL como source of truth para roles/permissions
3. Backend valida ID token + lookup no PostgreSQL
4. Custom Claims para cache de permissions (performance)

---

## Timeline Sugerido

| Fase | Duração | Sprint |
|-------|---------|--------|
| Fase 1: Configuração Firebase | 1 semana | Sprint 1 |
| Fase 2: Integração Backend | 1 semana | Sprint 2 |
| Fase 3: Apps Flutter | 2 semanas | Sprint 3-4 |
| Fase 4: Migração Usuários | 1 semana | Sprint 5 |
| Fase 5: Decomissionar JWT | 1 semana | Sprint 6 |

**Total estimado: 6 sprints (12 semanas)**

---

## Referências

- [Firebase Auth Custom Claims](https://firebase.google.com/docs/auth/admin/custom-claims)
- [Firebase Admin SDK](https://firebase.google.com/docs/admin/setup)
- [ADR-001: Filosofia de Arquitetura](../adr/ADR-001-nova-filosofia-arquitetura.md)
