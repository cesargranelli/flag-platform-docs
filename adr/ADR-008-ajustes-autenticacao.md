# ADR-008: Ajustes de Autenticação — Firebase Auth exclusivo + PENDING read-only

## Status
Aceito — implementado em `flag_backend@4482c5e` e `flag_admin_web@96b2c0d`.

## Contexto
ADR-004 definiu a migração para Firebase Auth + PostgreSQL como source of truth. Durante a implementação (Issues #39 e #82) surgiram novas decisões:
- `google-http-client 1.45.3 + JDK 25` causou `Not in GZIP format` em `verifyIdToken`/`createUser`.
- Tentativa de contorno com `FirebaseJwtVerifier` manual (java.net.http + jjwt) funcionou mas adicionou complexidade.
- Regra de negócio refinada: usuário `PENDING` deve conseguir logar e navegar em modo somente leitura, sem permissão de escrita.

## Decisão

### 1. Fluxo de autenticação
| Operação | Onde | Como |
|---|---|---|
| Signup | Flutter | `FirebaseAuth.signUpWithEmailPassword()` → `signOut()` → `POST /register {name,email}` |
| Login | Flutter | `FirebaseAuth.signInWithEmailPassword()` → `getIdToken(true)` → `GET /me` (Bearer ID Token) |
| Forgot password | Flutter | `FirebaseAuth.sendPasswordResetEmail()` direto (sem backend) |

Backend **não armazena nem valida senha**. Único ponto autenticado é `GET /me`.

### 2. Endpoints
- **Mantidos:** `POST /register` (público, sem senha), `GET /me`, `POST /users`, `GET /users`, `GET /users/pending`, `POST /users/{id}/approve|reject`.
- **Removidos:** `POST /login`, `POST /forgot-password`, `POST /reset-password`.

### 3. Modelo de dados
- `users.password_hash` removido (Migration V4 `V4__RemovePasswordHashAndResetTokens`).
- `password_reset_tokens` removido (mesma migration).
- `RegisterRequest` / `CreateUserRequest` sem campo `password`.
- `UserMapper`, `UserDetailsServiceImpl`, `StagingDataSeeder` sem referência a senha.

### 4. Verificação de token
- `FirebaseJwtVerifier` (verificação manual com `java.net.http` + `jjwt`) **removido**.
- `FirebaseTokenService` volta a usar `FirebaseAuth.verifyIdToken()` nativo do Firebase Admin SDK. O GZIP bug é contornado via `pom.xml` exclusion de `google-http-client-apache-v2` e `System.setProperty("java.net.preferIPv4Stack")` / `https.protocols` em `FirebaseConfig`.

### 5. PENDING read-only
- `JwtAuthenticationFilter` autentica **qualquer** status (antes bloqueava `PENDING`). Vincula `firebase_uid` via `AuthService.getOrProvisionFirebaseUser()`.
- `UserPrincipal` sempre `isEnabled=true`; authorities incluem `STATUS_ACTIVE` ou `STATUS_PENDING` + `ROLE_*`.
- `SecurityExpressions` (`ADMIN`, `ADMIN_OR_ORGANIZER`, `ADMIN_OR_MESA`, `CLUB_MANAGER`) agora exigem `hasAuthority('STATUS_ACTIVE')`. `PENDING` passa em `isAuthenticated()` e leitura (`GET`), mas falha em qualquer escrita (`POST/PUT/PATCH/DELETE`) com 403.

### 6. Frontend (flag_admin_web)
- `AuthApi`: removidos `loginWithFirebaseToken()`, `forgotPassword()`, `resetPassword()`; `register()` e `createUser()` sem `password`.
- `LoginResponse` e `reset_password_screen.dart` + rota `/reset-password` removidos; `forgot_password_screen.dart` mantido (usa Firebase SDK direto).
- `UserFormScreen` sem campo de senha.
- `app_router` sem import/rota de reset.

## Consequências
- Fluxo sem gambiarras: auth 100% Firebase SDK, backend é apenas lookup validado.
- PENDING tem UX melhor (vê dados, não fica bloqueado no login).
- Superfície de ataque reduzida (sem hash, sem reset tokens no banco).
- Dependência volta ao Firebase Admin SDK oficial — mais simples de manter.

## Arquivos chave
`flag_backend`: `AuthController`, `AuthService`, `RegisterRequest`, `UserEntity`, `SecurityConfig`, `JwtAuthenticationFilter`, `UserPrincipal`, `SecurityExpressions`, `FirebaseTokenService`, `FirebaseConfig`, `V4__RemovePasswordHashAndResetTokens`.
`flag_admin_web`: `auth_api.dart`, `signup_screen.dart`, `auth_controller.dart`, `user_form_screen.dart`, `app_router.dart`.

## Referências
- ADR-004 (migração Firebase Auth)
- `architecture/ajustes-autenticacao.md` (estado anterior, agora superado por este ADR)
- Spec PDF Seção 3 — Estratégia Híbrida
