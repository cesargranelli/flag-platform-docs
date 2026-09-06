# Reinstitution Plan for Flag Platform

## Strategic Directives
- Reinstitute Flag Platform from "moment zero" with a new architecture and data model.
- Preserving `main_v1` existant branches in repositories: `flag_admin_web`, `flag_backend`.
- A phases approach starting with `flag_admin_web`.
- Use combination of Flyway for DB migration management and JOOQ for type-safe persistence.
- Zero data in all tables before starting.
- Implement data migration logic for `organizations` -> `clubs` entity hierarchy change (ADR-003).

## Branching Strategy
- Preserve existing `main` branches.
- `develop` branch from current `main` for all repositories, to preserve history, where new development happens.
- `main_v1` branches from `develop` as a fallback mechanism.

## Phases

### Phase 0: Stabilize & Data Consolidation (Backend + Docs)
#### Goals
- Stabilize the core data model.
- Consolidation of the schema based on provided DDL.
- Prepare the backend for JOOQ.
- Create the branching structure for all repositories.

#### Step 0.1: Flag-platform-docs repo stabilization
- Done. This file is part of it.

#### Step 0.2: Branching for Rollback (Fase 0 - Fase de ramificação de rollback)
- DONE. Created and pushed `main_v1` branches from current `main` for `flag-platform-docs`, `flag_public_app`, `flag_referee_app`, `flag_tester_e2e`.

#### Step 0.3: Backend Stabilization & Data Consolidation (Fase 0 - Diagnóstico/Consolidação)
- Create `develop` branch on `flag_backend`.
- Clean old Flyway scripts on `develop` branch.
- Consolidate DDL to `V1__MomentZero.sql` adapting it for Flyway, ANSI SQL compliant.
- Configure `pom.xml` in `flag_backend` to include Flyway, JOOQ, and configure JOOQ plugin for type-safe generation mapping DB complex types to Java speculative enums.
- Implement zero data and data migration logic (`organizations` -> `clubs`) in Flyway scripts.

#### Step 0.4: Disable CI Pipelines
- Global disable of CI/CD pipelines to prevent premature deploys during phases 1-3.
- Update `flag_backend` CI pipeline to include JOOQ generation step.

### Phase 1: First decoupled app (`flag_admin_web`)
- Develop `flag_admin_web` features.
- Develop tests for `flag_admin_web`.
- Review and PR.
- Merge to `main`.
- Deploy.

### Phase 2: Mobile Referee App (`flag_referee_app`)
- Develop `flag_referee_app` features.
- Develop tests for `flag_referee_app`.
- Review and PR.
- Merge to `main`.

### Phase 3: Public App (`flag_public_app`)
- Develop `flag_public_app` features.
- Develop tests for `flag_public_app`.
- Review and PR.
- Merge to `main`.

### Phase 4: Global Integration & Quality Assurance (`flag_tester_e2e`)
- Develop cross-repo E2E tests in `flag_tester_e2e`.
- Review and PR.
- Merge to `main`.

### Phase 5: Release & Deployment
- Deploy Phase 1 (`flag_admin_web`).
- (Add more steps as needed)

### Phase 6: Future Deploys
- Deploy subsequent apps.

---

*This document is dynamic and will be updated as the project progresses. Check tasks state in TODO and report.*