# Ajustes no Módulo de Autenticação

> **Propósito:** Registrar as ações executadas para ajustar o módulo de autenticação da Flag Platform à estratégia híbrida descrita na Especificação Técnica (Seção 3), evitando perda de contexto em futuras sessões.

## 1. Estado Final (Concluído — Issue #39)

| Componente | Localização | Estado |
|---|---|---|
| `UserEntity.java` | `user/entity/` | ✅ Mapeia `firebase_uid`, `organization_id`, `club_id` |
| `UserRepository.java` | `user/repository/` | ✅ `findByFirebaseUid`, `existsByFirebaseUid` |
| `UserRole.java` | `common/enums/` | ✅ `ADMIN`, `ORGANIZER`, `MESA`, `ADMIN_LIGA`, `REFEREE`, `CLUB_MANAGER`, `FAN` |
| `UserResponse.java` | `user/dto/response/` | ✅ Inclui `firebaseUid`, `organizationId`, `clubId` |
| `UserPrincipal.java` | `security/` | ✅ Carrega `firebaseUid`, `organizationId`, `clubId` |
| `FirebaseTokenService.java` | `security/` | ✅ Valida token Firebase + fallback dev local |
| `FirebaseConfig.java` | `config/` | ✅ Inicialização resiliente do Firebase Admin SDK |
| `JwtAuthenticationFilter.java` | `security/` | ✅ Prioriza Firebase ID Token, fallback JWT local |
| `AuthService.java` | `user/service/` | ✅ `login()` valida Firebase ID Token, `getOrProvisionFirebaseUser`, `me(Object)` |
| `AuthController.java` | `user/controller/` | ✅ Login via Firebase, removidos `/forgot-password` e `/reset-password` |
| `LoginRequest.java` | `user/dto/request/` | ✅ Campo `firebaseIdToken` em vez de `email`+`password` |
| `SecurityConfig.java` | `config/` | ✅ `PUBLIC_AUTH_PATTERNS` contém apenas `/register` e `/login` |
| `pom.xml` (root) | raiz | ✅ `firebase-admin` 9.4.3 |

## 2. Alterações Executadas

### 2.1. Backend (`flag_backend` — Issue #39)

1. **`LoginRequest.java`** — Substituído `email` + `password` por `firebaseIdToken`.
2. **`AuthService.login()`** — Valida Firebase ID Token via `FirebaseTokenService`, procura ou provisiona usuário, gera JWT de sessão.
3. **`AuthController`** — Endpoints `/forgot-password` e `/reset-password` removidos. Login atualizado para aceitar `firebaseIdToken`.
4. **`SecurityConfig`** — `PUBLIC_AUTH_PATTERNS` removidos `/forgot-password` e `/reset-password`.
5. **`AuthService.register()`** — `passwordHash` definido como `null` (senhas não são armazenadas).
6. **`AuthService.createUser()`** — `passwordHash` definido como `null`.
7. **Build:** `mvn clean compile -DskipTests` → **BUILD SUCCESS**.

## 3. Documentação Relacionada

- **Especificação Técnica (PDF):** Seção 3 - Estratégia de Autenticação e Segurança Híbrida.
- **ADR-004:** `adr/ADR-004-firebase-auth-migration.md` — Migração de Autenticação para Firebase Auth + Custom Claims.
- **Plano de Migração:** `adr/plano-migracao-firebase-auth.md` — Fases detalhadas da migração JWT custom → Firebase Auth.
- **Arquitetura Híbrida:** `architecture/autenticacao-hibrida.md` — Diagrama de arquitetura, modelo de dados e ações de implementação.

## 4. Próximos Passos

1. ~~Executar ações pendentes da Issue #39 no `flag_backend`.~~ ✅ Concluído.
2. Implementar feature Clubs (Issue #82) no `flag_admin_web`.
3. Validar fluxo completo de autenticação ponta a ponta (Firebase Auth → Backend → PostgreSQL).
