# Arquitetura de Autenticação e Segurança Híbrida

> **Fonte:** Documento de Especificação Técnica do Ecossistema Flag Platform (Seção 3 - Estratégia de Autenticação e Segurança Híbrida).

## 1. Visão Geral

Para garantir alto padrão de segurança, rapidez no desenvolvimento de logins sociais (Google, Apple) e leveza no servidor, a Flag Platform adota uma arquitetura híbrida de autenticação:

* **Firebase Auth como Provedor de Identidade (IDP):** Cuida exclusivamente do fluxo de identidade (*"Quem é você"*), validação de credenciais, redefinições de senha via e-mail e emissão/renovação de tokens criptografados JWT (Firebase ID Tokens).
* **Flag Backend & SQL:** O banco de dados relacional **não armazena hashes de senhas** nem gerencia fluxos de recuperação de senha locais. A tabela de usuários possui uma chave única de amarração chamada **`firebase_uid`**. O backend atua exclusivamente na governança e autorização (*"O que você pode fazer"*), validando a assinatura e claims do JWT enviado nas requisições.

---

## 2. Diagrama de Arquitetura de Solução

```mermaid
graph TD
    A[Flag Admin Web]
    B[Flag Referee App]
    C[Flag Public App]
    FA[Firebase Auth]
    BE[Flag Backend]
    DB[(PostgreSQL)]
    TE[Flag Tester e2e]

    %% --- FLUXO 1: AUTENTICACAO ---
    A -.->|1. Envia credenciais / OAuth| FA
    B -.->|1. Envia credenciais / OAuth| FA
    C -.->|1. Envia credenciais / OAuth| FA
    FA -.->|2. Retorna Token JWT (ID Token)| A
    FA -.->|2. Retorna Token JWT (ID Token)| B
    FA -.->|2. Retorna Token JWT (ID Token)| C

    %% --- FLUXO 2: REQUISICOES AUTENTICADAS ---
    A ==>|3. API Request + Authorization: Bearer JWT| BE
    B ==>|3. API Request + Authorization: Bearer JWT| BE
    C ==>|3. API Request + Authorization: Bearer JWT| BE

    %% --- FLUXO 3: VALIDACAO INTERNA E BANCO ---
    BE -->|4. Valida assinatura / emissor do Token| FA
    BE ==>|5. Busca perfil por firebase_uid| DB

    %% --- COMUNICACAO DE DADOS ---
    BE <-->|Leitura e Escrita| DB

    %% --- CONEXAO DA ESTEIRA DE TESTES ---
    TE ==>|Simula jornadas| A
    TE ==>|Simula súmula| B
    TE ==>|Valida dados ao vivo| C
```

---

## 3. Modelo de Dados Relacional (`users`)

No banco de dados relacional (PostgreSQL), a tabela `platform.users` reflete o perfil e os relacionamentos institucionais:

```sql
CREATE TABLE platform.users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firebase_uid VARCHAR(255) NOT NULL UNIQUE,
    organization_id UUID NULL REFERENCES platform.organizations(id),
    club_id UUID NULL REFERENCES platform.clubs(id),
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    role VARCHAR(50) NOT NULL, -- "ADMIN_LIGA" | "REFEREE" | "CLUB_MANAGER" | "FAN"
    status VARCHAR(50) NOT NULL DEFAULT 'ACTIVE',
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);
```

### Regras Críticas:
1. **Sem Senhas no Banco:** Não há coluna `password_hash` nem tabelas como `password_reset_tokens`.
2. **Chave Externa:** O campo `firebase_uid` é o elo unívoco entre a conta no Firebase e os dados de domínio no PostgreSQL.
3. **Roles e Autorização:**
   * `ADMIN_LIGA`: Administrador/organizador da liga/federação (acesso ao Flag Admin Web).
   * `REFEREE`: Árbitros e oficiais de mesa (acesso ao Flag Referee App).
   * `CLUB_MANAGER`: Dirigente/gestor de clube específico.
   * `FAN`: Torcedor / atleta / público geral (acesso ao Flag Public App).

---

## 4. Ações de Implementação e Ajuste

### 4.1. Flag Backend (`flag_backend` - Issue #39)

1. **Entidade e Repositório:**
   * Mapear `firebase_uid` (`@Column(name = "firebase_uid", unique = true)`) e `organization_id` em `UserEntity.java`.
   * Adicionar `findByFirebaseUid(String firebaseUid)` em `UserRepository.java`.
   * Atualizar `UserRole.java` para acomodar `ADMIN_LIGA`, `REFEREE`, `CLUB_MANAGER`, `FAN` mantendo compatibilidade com os papéis vigentes.
2. **Validação de Token (Spring Security):**
   * Configurar `FirebaseAuth` via Firebase Admin SDK (ou validação JWT RS256 com chaves públicas do Google).
   * Implementar `FirebaseAuthenticationFilter` extraindo o token do header `Authorization: Bearer <idToken>`.
   * Extrair o `uid` e e-mail do token e injetar no `SecurityContextHolder`.
3. **Endpoints de Auth:**
   * Desativar `POST /api/v1/auth/login` (login local com conferência de hash).
   * Consolidar `GET /api/v1/auth/me` para recuperar os dados a partir do `firebase_uid` do usuário autenticado no contexto de segurança.
   * Auto-provisionamento/vínculo: caso um usuário autenticado no Firebase ainda não tenha registro local, provisionar perfil inicial ou vincular por e-mail.
   * Remover endpoints locais de recuperação de senha (`/forgot-password`, `/reset-password`).

### 4.2. Flag Admin Web (`flag_admin_web` - Issue #33 / #82)

1. **Autenticação no Cliente:**
   * Utilizar `FirebaseAuthService.signInWithEmailPassword` (ou Google/Apple).
   * Obter o ID Token atualizado via `user.getIdToken()`.
2. **Injeção de Headers HTTP:**
   * `ApiClient` injeta automaticamente `Authorization: Bearer <idToken>` em todas as requisições autenticadas.
3. **Recuperação de Senha:**
   * Realizada diretamente pelo cliente via `FirebaseAuth.instance.sendPasswordResetEmail(email)`.
4. **Resolução de Sessão:**
   * Após login no Firebase, chamar `GET /api/v1/auth/me` para obter permissões, `role` e `organizationId` do usuário no backend.
