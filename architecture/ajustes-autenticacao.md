# Ajustes no Módulo de Autenticação

> **Propósito:** Registrar as ações necessárias para ajustar o módulo de autenticação da Flag Platform à estratégia híbrida descrita na Especificação Técnica (Seção 3), evitando perda de contexto em futuras sessões.

## 1. Estado Atual (Levantamento)

| Componente | Localização | Estado |
|---|---|---|
| `UserEntity.java` | `user/entity/` | ✅ Já mapeia `firebase_uid`, `organization_id`, `club_id` |
| `UserRepository.java` | `user/repository/` | ✅ Já tem `findByFirebaseUid` e `existsByFirebaseUid` |
| `UserRole.java` | `common/enums/` | ✅ Já tem `ADMIN_LIGA`, `REFEREE`, `CLUB_MANAGER`, `FAN` |
| `UserResponse.java` | `user/dto/response/` | ✅ Já inclui `firebaseUid`, `organizationId`, `clubId` |
| `UserPrincipal.java` | `security/` | ✅ Já carrega `firebaseUid`, `organizationId`, `clubId` |
| `FirebaseTokenService.java` | `security/` | ✅ Valida token Firebase + fallback dev |
| `FirebaseConfig.java` | `config/` | ✅ Inicialização resiliente do Firebase Admin SDK |
| `JwtAuthenticationFilter.java` | `security/` | ✅ Prioriza Firebase ID Token, fallback JWT local |
| `AuthService.java` | `user/service/` | ✅ Já tem `getOrProvisionFirebaseUser` e `me(Object)` |
| `pom.xml` (root) | raiz | ✅ Dependência `firebase-admin` 9.4.3 já presente |
| `FirebaseUserInfo.java` | `security/` | ✅ DTO com uid, email, name, claims |

## 2. Ações Pendentes / Ajustes Necessários

### 2.1. Backend (`flag_backend` — Issue #39)

1. **Remover/desativar endpoints legados de senha:**
   - `POST /api/v1/auth/login` (login com password_hash local) → substituir por validação de Firebase ID Token.
   - `POST /api/v1/auth/forgot-password` → remover (responsabilidade do Firebase Auth).
   - `POST /api/v1/auth/reset-password` → remover (responsabilidade do Firebase Auth).
   - `POST /api/v1/auth/register` → reavaliar: cadastro direto no backend vs. auto-provisionamento via Firebase + `/auth/me`.

2. **Consolidar `GET /api/v1/auth/me`:**
   - Recuperar usuário autenticado pelo `firebase_uid` do contexto de segurança.
   - Se usuário não existir no banco, auto-provisionar (já implementado em `getOrProvisionFirebaseUser`).
   - Retornar `UserResponse` com `firebaseUid`, `organizationId`, `clubId`, `role`, `status`.

3. **Ajustar `SecurityConfig.java`:**
   - Remover `/api/v1/auth/login`, `/api/v1/auth/forgot-password`, `/api/v1/auth/reset-password` de `PUBLIC_AUTH_PATTERNS` (ou manter como legado temporário com deprecation).
   - Garantir que `PUBLIC_AUTH_PATTERNS` contenha apenas endpoints válidos para a nova estratégia.

4. **Testes de integração:**
   - Validar que requisições com `Authorization: Bearer <firebase_id_token>` são autenticadas corretamente.
   - Validar fallback dev (sem credenciais Firebase) não quebra o startup.
   - Executar `mvn clean compile -DskipTests`.

### 2.2. Frontend (`flag_admin_web` — Issues #33 / #82)

1. **Manter `FirebaseAuthService` como IDP exclusivo:**
   - `signInWithEmailPassword`, `signInWithPopup` (Google/Apple).
   - `getIdToken()` para obter o JWT a enviar no header.

2. **`ApiClient`:**
   - Injetar `Authorization: Bearer <idToken>` em todas as requisições autenticadas (já implementado).

3. **`AuthController` (frontend):**
   - After Firebase login → chamar `GET /api/v1/auth/me` para obter perfil completo (role, organizationId, clubId).
   - Persistir sessão Firebase para restauração futura.

4. **Recuperação de senha:**
   - Chamar `FirebaseAuth.instance.sendPasswordResetEmail(email)` diretamente do frontend (sem endpoint backend).

5. **Remover referência a `packages/`:**
   - Pasta já removida. Garantir que não há imports quebrados (já validado com `flutter analyze`).

## 3. Documentação Relacionada

- **Especificação Técnica (PDF):** Seção 3 - Estratégia de Autenticação e Segurança Híbrida.
- **ADR-004:** `adr/ADR-004-firebase-auth-migration.md` — Migração de Autenticação para Firebase Auth + Custom Claims.
- **Plano de Migração:** `adr/plano-migracao-firebase-auth.md` — Fases detalhadas da migração JWT custom → Firebase Auth.
- **Arquitetura Híbrida:** `architecture/autenticacao-hibrida.md` — Diagrama de arquitetura, modelo de dados e ações de implementação.

## 4. Próximos Passos

1. Executar ações pendentes da Issue #39 no `flag_backend`.
2. Implementar feature Clubs (Issue #82) no `flag_admin_web`.
3. Validar fluxo completo de autenticação ponta a ponta (Firebase Auth → Backend → PostgreSQL).
