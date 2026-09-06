# ADR-007: Migrações Flyway em Java Utilizando a DSL do JOOQ

## Status
Aprovado

## Contexto
Durante a reinstituição da Flag Platform a partir do Momento Zero (Fase 0), foi estabelecido o uso conjunto de Flyway para controle de versão de banco de dados e JOOQ para persistência type-safe.

Tradicionalmente, migrações Flyway podem ser escritas em arquivos SQL puros (`.sql`) ou em classes Java (`BaseJavaMigration`). O uso exclusivo de SQL puro em migrações complexas ou de dados apresenta os seguintes desafios:
- Falta de tipagem e verificação em tempo de compilação.
- Risco de regressão sintática entre diferentes dialetos ou alterações de schema.
- Dificuldade em reutilizar enums e tipos gerados pelo JOOQ.
- Tratamento complexo de lógica condicional ou transformações de dados (como a separação de `organizations` para `clubs` da ADR-003).

## Decisão
A partir do baseline estabelecido no `V1__MomentZero.sql` (schema inicial de referência):
1. **Todas as novas migrações Flyway serão implementadas em classes Java** estendendo `org.flywaydb.core.api.migration.BaseJavaMigration`.
2. As operações de DDL e DML dentro dessas migrações serão escritas **utilizando a DSL type-safe do JOOQ** (`DSLContext` / `DSL`).
3. O pacote padrão de localização das migrações Java será `db.migration` (sob `src/main/java/db/migration`), padrão reconhecido automaticamente pelo Flyway.
4. Nomenclatura das classes: seguir a convenção Flyway Java (`V{n}__{DescricaoEmPascalCase}.java`), por exemplo: `V2__CreateClubsAndMigrateOrganizations.java`.

## Consequências
### Positivas
- **Type-Safety Total:** Erros de colunas, tabelas ou tipos são capturados em tempo de compilação.
- **Lógica e Transformação Expressiva:** Manipulações de dados (como migração de `organizations` para `clubs`, sanitização, criptografia ou geração de hashes) contam com todo o poder da linguagem Java e do ecossistema Spring/JOOQ.
- **Integração Fluida:** Código gerado pelo JOOQ a partir da baseline pode ser consumido diretamente nas migrações seguintes.

### Negativas / Mitigações
- Exige que o código do projeto compile antes da execução das migrações (mitigado pelo fluxo padrão de build Maven/CI).
- Maior verbosidade em alterações simples de DDL (mitigado pela legibilidade e segurança da DSL do JOOQ).
