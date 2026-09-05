# ADR-003: Implementação da Hierarquia de 5 Níveis (Club Entity)

## Contexto

A Flag Platform precisa suportar uma hierarquia organizacional clara para o ecossistema de Flag Football no Brasil:

```
Organization (Federação/Liga)
    └── Club (Clube/Universidade)
         └── Team (Time de competição)
              └── Roster (Elenco por Season)
                   └── Athlete (Atleta)
```

**Problema atual:** O schema atual usa `Organization` para representar tanto Federações/Ligas quanto Clubes, utilizando um discriminator `organization_type = 'CLUB'` ou `'UNIVERSITY'`. Isso causa:
- Conflação de conceitos (Federação ≠ Clube)
- Impossibilidade de enforce FK hierárquico correto
- Dificuldade em separar permissões por nível

## Decisão

**Criar uma tabela `clubs` dedicada** que separa Clubes de Organizações, permitindo:

1. FK correto: `Organization → Club → Team → Roster → Athlete`
2. Clubes pertencem a uma Organization (Federação/Liga)
3. Equipes pertencem a um Club
4. Permissões granulares por nível hierárquico

## Implementação

### 1. Nova Tabela `clubs`

```sql
CREATE TABLE platform.clubs
(
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id   UUID NOT NULL,
    name             VARCHAR(255) NOT NULL,
    short_name       VARCHAR(50),
    sport_name       VARCHAR(255),
    logo_url         VARCHAR(500),
    document         VARCHAR(20),
    document_type    VARCHAR(10),
    president_name  VARCHAR(150),
    president_cpf   VARCHAR(14),
    status           VARCHAR(20) DEFAULT 'ACTIVE',
    created_at       TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at       TIMESTAMP,
    created_by       UUID,
    updated_by       UUID,
    CONSTRAINT fk_clubs_organization FOREIGN KEY (organization_id) 
        REFERENCES platform.organizations (id)
);
```

### 2. Migração de Dados

Organizações existentes com `organization_type IN ('CLUB', 'UNIVERSITY')` serão migradas para a nova tabela `clubs`.

### 3. Refatoração de `team`

A tabela `team` terá seu FK alterado de `organization_id` para `club_id`.

**Antes:**
```sql
ALTER TABLE platform.team ADD COLUMN club_id UUID;
UPDATE platform.team SET club_id = organization_id;
ALTER TABLE platform.team DROP COLUMN organization_id;
```

## Flyway Migrations

| Versão | Descrição |
|--------|-----------|
| V38 | Criar tabela `clubs` |
| V39 | Migrar dados de Clubes de `organizations` para `clubs` |
| V40 | Adicionar `club_id` em `team` e remover `organization_id` |
| V41 | (Code-only) Criar módulo `br.com.flagplatform.club` |

## Novo Módulo Java: `club`

```
src/main/java/br/com/flagplatform/club/
├── Club.java                  # @ApplicationModule
├── ClubLookup.java            # Interface pública
├── ClubInfo.java             # Record de projeção
├── controller/
│   └── ClubController.java   # CRUD REST
├── service/
│   └── ClubService.java      # Regras de negócio
├── repository/
│   └── ClubRepository.java   # Spring Data JPA
├── entity/
│   └── ClubEntity.java       # JPA Entity
├── mapper/
│   └── ClubMapper.java       # MapStruct
├── dto/
│   ├── request/
│   │   └── CreateClubRequest.java
│   └── response/
│       └── ClubResponse.java
└── exception/
    ├── ClubNotFoundException.java
    └── DuplicateClubException.java
```

## Regras de Negócio

1. **Criação de Club:** Apenas `SUPER_ADMIN` ou `ORG_ADMIN` vinculado à Organization pode criar Clubs.
2. **Edição de Club:** Apenas `ORG_ADMIN` vinculado ao Club pode editar.
3. **Listagem de Clubs:** Acesso público via `GET /api/v1/organizations/{id}/clubs`.

## Alternativas Consideradas

### A. Manter Organization como container único
- **Prós:** Sem mudança de schema
- **Contras:** Conflação de conceitos, FK ambíguo, permissões confusas

### B. Usar `parent_id` na Organization para hierarquia
- **Prós:** Não requer nova tabela
- **Contras:** Mesmos problemas de separação conceitual, queries mais complexas

### C. Firestore-first com Clubs como subcollection
- **Prós:** Schema flexível
- **Contras:** Perde benefícios de PostgreSQL (joins, FK, integridade)

---

**Status**: Proposto  
**Data**: 2026-09-06  
**Autor**: Tech Lead (Flag Platform)
