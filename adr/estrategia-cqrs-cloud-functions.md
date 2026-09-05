# ADR-007: Estratégia de Espelhamento CQRS com Cloud Functions

**Status:** Proposto
**Data:** 2026-09-05
**Autor:** DevOps (Flag Platform)
**Substitui/Complementa:** ADR-002 (PostgreSQL + Firestore CQS)

---

## Contexto

A Flag Platform adota **CQRS Light**: PostgreSQL é a *source of truth* (write path) e Firestore é o *espelho de leitura* (read path) para dashboards, leaderboards, real-time scores e casos onde Firestore oferece melhor performance/custo.

Conforme definido em [ADR-002](ADR-002-postgres-firestore-cqs.md), a sincronização entre PostgreSQL e Firestore é o ponto crítico da arquitetura. Este ADR define **como** essa sincronização acontece na prática — quais triggers, Cloud Functions, schema mapping, error handling e monitoramento.

### Restrições
- Volume baixo (~10-20 TPS) → sem necessidade de event broker dedicado (Kafka, Pub/Sub)
- Backend Spring Boot (Modular Monolith) é o único produtor de escritas em PostgreSQL
- Firestore é consumido por apps (Public App, dashboards) via SDKs nativos
- Cloud Functions Gen 2 (Node.js 20) já suportadas pelo ambiente

---

## Decisão

Adotar **três camadas complementares** de sincronização, priorizadas por confiabilidade e simplicidade operacional:

| Camada | Trigger | Função | Latência esperada | Uso |
|--------|---------|--------|-------------------|-----|
| **Primária** | HTTP trigger (backend → Cloud Function) | Sync imediato após commit | 100-500ms | 95% dos casos |
| **Reativa** | Firestore onWrite (client SDK direto) | Sync fallback quando backend não chama | 1-3s | Edge cases, escritas offline |
| **Reconciliação** | Cloud Scheduler (cron) | Full/partial rebuild | 5-15min | Recuperação, drift correction |

### Por quê três camadas?

1. **HTTP trigger** garante que toda escrita do backend dispare sincronização explícita (auditoria, retry previsível, controle de versão).
2. **Firestore onWrite** cobre cenários onde o client escreve direto no Firestore (otimizações mobile-first, offline) e serve como *safety net*.
3. **Cloud Scheduler** é a rede de segurança para corrigir drift causado por falhas transitórias, sem intervenção manual.

---

## Arquitetura de Sincronização

```
┌──────────────────────────────────────────────────────────────────────┐
│                          WRITE PATH (Source of Truth)                │
│                                                                      │
│  ┌─────────────┐                                                     │
│  │ Spring Boot │                                                     │
│  │   Backend   │                                                     │
│  │ (Modular    │                                                     │
│  │  Monolith)  │                                                     │
│  └──────┬──────┘                                                     │
│         │ 1. Transação ACID                                          │
│         ▼                                                            │
│  ┌──────────────┐                                                    │
│  │  PostgreSQL  │                                                    │
│  │ (Cloud SQL)  │                                                    │
│  └──────┬───────┘                                                    │
│         │ 2. After commit hook (ApplicationEventPublisher)          │
│         ▼                                                            │
│  ┌────────────────────────┐                                          │
│  │ CqrsSyncService (Java) │                                          │
│  │ - Coleta evento        │                                          │
│  │ - Enriquecer payload   │                                          │
│  │ - HTTP POST assíncrono │                                          │
│  └──────────┬─────────────┘                                          │
└─────────────┼────────────────────────────────────────────────────────┘
             │ HTTPS
             ▼
┌──────────────────────────────────────────────────────────────────────┐
│                     SYNC LAYER (Cloud Functions)                     │
│                                                                      │
│  ┌─────────────────────────────────────────────────────┐             │
│  │ syncEntity (Gen 2, HTTP trigger)                    │             │
│  │ - Valida payload (JWT service-to-service)            │             │
│  │ - Mapeia schema PostgreSQL → Firestore               │             │
│  │ - Batch write em Firestore (atomic por entidade)     │             │
│  │ - Publica evento de sucesso/erro no logs             │             │
│  └──────────────────────┬──────────────────────────────┘             │
│                         │                                            │
│  ┌──────────────────────┴──────────────────────────────┐             │
│  │ firestoreMirror (Gen 2, Firestore onWrite trigger)  │             │
│  │ - Safety net: detecta escritas que não vieram do     │             │
│  │   backend e revalida contra PostgreSQL              │             │
│  │ - Apenas READ-ONLY: nunca escreve em PostgreSQL     │             │
│  └──────────────────────┬──────────────────────────────┘             │
│                         │                                            │
│  ┌──────────────────────┴──────────────────────────────┐             │
│  │ reconcileFirestore (Gen 2, Cloud Scheduler)         │             │
│  │ - Cron a cada 15min                                 │             │
│  │ - Compara updated_at PostgreSQL vs Firestore        │             │
│  │ - Reaplica divergências                             │             │
│  │ - Marca entidades órfãs (deleted_at != null)        │             │
│  └─────────────────────────────────────────────────────┘             │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────────────┐
│                          READ PATH (Espelho)                         │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │ Public App   │  │ Admin Web    │  │ Referee App  │               │
│  │ (Flutter)    │  │ (Flutter Web)│  │ (Flutter)    │               │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘               │
│         │                 │                 │                        │
│         └─────────────────┼─────────────────┘                        │
│                           ▼                                          │
│                  ┌─────────────────┐                                │
│                  │   Firestore     │                                │
│                  │   (Collections) │                                │
│                  └─────────────────┘                                │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Fluxo de Dados Detalhado (Write → Read)

### 1. Escrita no Backend

```java
// Service genérico (exemplo: TeamService)
@Service
public class TeamService {
    private final CqrsSyncService cqrsSync;
    
    @Transactional
    public Team update(UUID id, TeamDTO dto) {
        Team team = teamRepository.save(mapper.toEntity(dto));
        
        // 1. Commit PostgreSQL (ACID garantido)
        // 2. Após commit, dispara sync (assíncrono, não bloqueia response)
        cqrsSync.publish("teams", team.getId(), SyncOperation.UPDATE, team);
        
        return team;
    }
}
```

### 2. Hook de Pós-Commit

```java
// CqrsSyncService - usa ApplicationEventPublisher (não bloqueia)
@Service
public class CqrsSyncService {
    private final RestTemplate http;
    private final String syncUrl; // Cloud Function URL
    
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onSyncEvent(SyncEvent event) {
        try {
            http.postForEntity(
                syncUrl + "/" + event.getCollection(),
                new SyncPayload(event),
                Void.class
            );
        } catch (Exception e) {
            // Falha aqui → reconciliação corrige depois (drift tolerance)
            log.warn("Sync failed for {}:{} - reconciler will retry", 
                event.getCollection(), event.getId(), e);
        }
    }
}
```

### 3. Cloud Function HTTP Trigger

```javascript
// functions/src/syncEntity.ts (Gen 2)
import { onRequest } from "firebase-functions/v2/https";
import { getFirestore, FieldValue } from "firebase-admin/firestore";
import { logger } from "firebase-functions/v2";
import { authenticateService } from "./auth";
import { mapEntity } from "./mappers";

const db = getFirestore();

export const syncEntity = onRequest(
  { region: "southamerica-east1", cors: false, secrets: ["INTERNAL_API_KEY"] },
  async (req, res) => {
    // 1. Autenticação service-to-service
    if (!authenticateService(req)) {
      res.status(401).send("Unauthorized");
      return;
    }

    const { collection, id, operation, payload, version } = req.body;

    try {
      // 2. Validação de schema
      if (!collection || !id || !operation) {
        res.status(400).send("Missing required fields");
        return;
      }

      // 3. Mapeamento PostgreSQL → Firestore
      const mapped = mapEntity(collection, payload);

      // 4. Escrita idempotente (version check)
      const ref = db.collection(collection).doc(id);
      await db.runTransaction(async (tx) => {
        const current = await tx.get(ref);
        const currentVersion = current.data()?.version ?? 0;
        if (version <= currentVersion && operation !== "DELETE") {
          logger.warn(`Stale write for ${collection}/${id} v${version} <= v${currentVersion}`);
          return; // Idempotente: ignora versão antiga
        }

        if (operation === "DELETE") {
          tx.set(ref, { deleted: true, deletedAt: FieldValue.serverTimestamp(), version }, { merge: true });
        } else {
          tx.set(ref, { ...mapped, version, updatedAt: FieldValue.serverTimestamp() }, { merge: true });
        }
      });

      logger.info(`Synced ${collection}/${id} v${version} op=${operation}`);
      res.status(200).send({ ok: true });
    } catch (err) {
      logger.error(`Sync failed ${collection}/${id}`, err);
      res.status(500).send({ error: err.message });
      // Não relança → deixa reconciler pegar
    }
  }
);
```

### 4. Firestore onWrite (Safety Net)

```javascript
// functions/src/firestoreMirror.ts
import { onDocumentWritten } from "firebase-functions/v2/firestore";
import { getFirestore } from "firebase-admin/firestore";

const db = getFirestore();

export const firestoreMirror = onDocumentWritten(
  { region: "southamerica-east1", document: "{collection}/{id}" },
  async (event) => {
    const { collection, id } = event.params;
    const after = event.data?.after.data();

    // Ignora coleções que não fazem parte do espelho CQRS
    if (!MIRRORED_COLLECTIONS.includes(collection)) return;

    // Ignora se foi escrito pelo próprio sync (tem campo _syncedBy)
    if (after?._syncedBy === "syncEntity") return;

    logger.warn(`Firestore onWrite triggered for ${collection}/${id} - validating against PostgreSQL`);

    // Re-busca no PostgreSQL e reescreve (corrige drift)
    const pgData = await fetchFromPostgres(collection, id);
    if (pgData) {
      await db.collection(collection).doc(id).set(
        { ...mapEntity(collection, pgData), _syncedBy: "firestoreMirror" },
        { merge: true }
      );
    } else {
      // Não existe no PostgreSQL → marca como deletado
      await db.collection(collection).doc(id).set(
        { deleted: true, deletedAt: FieldValue.serverTimestamp(), _syncedBy: "firestoreMirror" },
        { merge: true }
      );
    }
  }
);
```

### 5. Reconciliação (Cloud Scheduler)

```javascript
// functions/src/reconcileFirestore.ts
import { onSchedule } from "firebase-functions/v2/scheduler";
import { getFirestore } from "firebase-admin/firestore";
import { Client } from "pg";

const db = getFirestore();
const pg = new Client({ connectionString: process.env.PG_CONNECTION });

export const reconcileFirestore = onSchedule(
  { schedule: "every 15 minutes", region: "southamerica-east1", timeZone: "America/Sao_Paulo" },
  async () => {
    for (const collection of MIRRORED_COLLECTIONS) {
      const since = await getLastReconcileTime(collection);
      const changed = await fetchFromPostgres(collection, { updatedAfter: since });

      const batch = db.batch();
      for (const row of changed) {
        const ref = db.collection(collection).doc(row.id);
        batch.set(ref, mapEntity(collection, row), { merge: true });
      }
      await batch.commit();

      await setLastReconcileTime(collection, new Date());
      logger.info(`Reconciled ${changed.length} ${collection} since ${since.toISOString()}`);
    }
  }
);
```

---

## Cloud Functions Necessárias

| Função | Trigger | Runtime | Região | Concurrency | Memória | Timeout |
|--------|---------|---------|--------|-------------|---------|---------|
| `syncEntity` | HTTPS | Node.js 20 | southamerica-east1 | 80 | 512MiB | 60s |
| `syncEntityBatch` | HTTPS (bulk) | Node.js 20 | southamerica-east1 | 40 | 1GiB | 300s |
| `firestoreMirror` | Firestore onWrite | Node.js 20 | southamerica-east1 | 100 | 256MiB | 30s |
| `reconcileFirestore` | Cloud Scheduler | Node.js 20 | southamerica-east1 | 1 | 1GiB | 540s |
| `reconcileFull` | Cloud Scheduler (diário) | Node.js 20 | southamerica-east1 | 1 | 2GiB | 900s |
| `purgeDeleted` | Cloud Scheduler (semanal) | Node.js 20 | southamerica-east1 | 1 | 512MiB | 300s |

### Responsabilidades por Função

- **`syncEntity`** — Sincronização unitária (1 entidade). Recebe `{collection, id, operation, payload, version}`. Idempotente via version check.
- **`syncEntityBatch`** — Sincronização em lote (até 500 entidades por chamada). Usado em migrações iniciais e backfills.
- **`firestoreMirror`** — Detecta escritas diretas no Firestore (não vieram do backend) e revalida contra PostgreSQL.
- **`reconcileFirestore`** — Incremental: compara `updated_at` desde última execução. Roda a cada 15min.
- **`reconcileFull`** — Full rebuild: recarrega todas as entidades. Roda 1x/dia às 03:00 (janela de baixo tráfego).
- **`purgeDeleted`** — Remove de Firestore entidades com `deleted_at < now() - 90d`. Compliance LGPD.

---

## Schema Mapping PostgreSQL → Firestore

### Convenções

- **IDs:** UUID (string) preservado em ambos os lados
- **Timestamps:** `TIMESTAMPTZ` → Firestore `Timestamp` (com `_seconds`/`_nanoseconds` no JSON)
- **Enums:** PostgreSQL `VARCHAR` + CHECK → Firestore `string` validado por Security Rules
- **JSONB:** PostgreSQL → Firestore `Map`
- **Relacionamentos:** FK ID preservado como `string`; subcoleções quando 1:N com forte acoplamento
- **Soft delete:** `deleted_at TIMESTAMPTZ NULL` → campo `deleted: boolean` + `deletedAt: Timestamp`
- **Versionamento:** Toda entidade tem `version BIGINT` (PostgreSQL) ↔ `version: number` (Firestore)

### Mapeamento por Entidade

| Entidade | PostgreSQL Table | Firestore Collection | Doc ID | Subcollections | Observações |
|----------|------------------|----------------------|--------|----------------|-------------|
| Organization | `organizations` | `organizations` | UUID | `clubs`, `competitions` | Nome, slug, logo, country |
| Competition | `competitions` | `competitions` | UUID | `categories`, `rounds` | season, status, dates |
| Category | `categories` | `categories` | UUID | `teams`, `standings` | modality, gender, age range |
| Modality | `modalities` | `modalities` | UUID | — | Catálogo (5x5, 8x8, 9x9) |
| Venue | `venues` | `venues` | UUID | `games` (data denormalizado) | address, geo |
| Club | `clubs` | `clubs` | UUID | `teams` | Pertence a Organization |
| Team | `teams` | `teams` | UUID | `roster_entries` | Pertence a Club + Category |
| Athlete | `athletes` | `athletes` | UUID | — | profile, skills, photo URL |
| RosterEntry | `team_roster` | `roster_entries` | UUID | — | teamId, athleteId, season |
| Round | `rounds` | `rounds` | UUID | `games` | categoryId, dates |
| Game | `games` | `games` | UUID | `score_events` | roundId, homeTeamId, awayTeamId |
| ScoreEvent | `score_events` | `score_events` | UUID | — | gameId, athleteId, type, points |
| Standing | `standings` | `standings` | UUID | — | categoryId, teamId, wins, points |
| CheckIn | `checkins` | `checkins` | UUID | — | gameId, athleteId, status, timestamp |
| User | `users` | `users` | UUID | — | Apenas dados não-sensíveis (id, role) |
| User (privados) | `users` | (NÃO espelhado) | — | — | email, passwordHash, tokens → ficam só no PostgreSQL |

### Exemplo de Mapeamento (Team)

**PostgreSQL (`teams` table):**
```sql
CREATE TABLE teams (
    id UUID PRIMARY KEY,
    club_id UUID NOT NULL REFERENCES clubs(id),
    category_id UUID NOT NULL REFERENCES categories(id),
    name VARCHAR(120) NOT NULL,
    short_name VARCHAR(40),
    logo_url TEXT,
    primary_color VARCHAR(7),
    secondary_color VARCHAR(7),
    coach_id UUID,
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMPTZ,
    version BIGINT NOT NULL DEFAULT 1
);
```

**Firestore (`teams/{id}` document):**
```typescript
{
  id: string,                    // UUID
  clubId: string,                // FK
  categoryId: string,            // FK
  name: string,
  shortName: string | null,
  logoUrl: string | null,
  primaryColor: string | null,   // "#RRGGBB"
  secondaryColor: string | null,
  coachId: string | null,        // FK
  active: boolean,
  createdAt: Timestamp,
  updatedAt: Timestamp,
  deletedAt: Timestamp | null,
  deleted: boolean,
  version: number,               // otimistic concurrency
  _syncedBy: string,             // "syncEntity" | "firestoreMirror" | "reconcileFirestore"
  _syncedAt: Timestamp
}
```

**Mapeador (TypeScript):**
```typescript
// functions/src/mappers/team.ts
export const mapTeam = (row: TeamRow): TeamDoc => ({
  id: row.id,
  clubId: row.club_id,
  categoryId: row.category_id,
  name: row.name,
  shortName: row.short_name,
  logoUrl: row.logo_url,
  primaryColor: row.primary_color,
  secondaryColor: row.secondary_color,
  coachId: row.coach_id,
  active: row.active,
  createdAt: toTimestamp(row.created_at),
  updatedAt: toTimestamp(row.updated_at),
  deletedAt: row.deleted_at ? toTimestamp(row.deleted_at) : null,
  deleted: row.deleted_at !== null,
  version: Number(row.version),
  _syncedBy: "syncEntity",
  _syncedAt: FieldValue.serverTimestamp(),
});
```

---

## Índices Firestore Necessários

| Collection | Campos indexados | Tipo | Query que atende |
|------------|------------------|------|------------------|
| `organizations` | `deleted ASC, name ASC` | Composite | `where('deleted', '==', false).orderBy('name')` |
| `competitions` | `organizationId ASC, status ASC, startDate DESC` | Composite | Dashboard de competições ativas por org |
| `categories` | `competitionId ASC, modality ASC, gender ASC` | Composite | Filtros de categorias |
| `teams` | `clubId ASC, active ASC, name ASC` | Composite | Listagem de times por clube |
| `teams` | `categoryId ASC, active ASC` | Composite | Times por categoria |
| `athletes` | `deleted ASC, name ASC` | Composite | Listagem geral |
| `roster_entries` | `teamId ASC, seasonId ASC` | Composite | Elenco por time/temporada |
| `rounds` | `categoryId ASC, startDate ASC` | Composite | Calendário por categoria |
| `games` | `roundId ASC, scheduledAt ASC` | Composite | Jogos por rodada |
| `games` | `homeTeamId ASC, status ASC`, `awayTeamId ASC, status ASC` | Composite | Histórico por time |
| `games` | `status ASC, scheduledAt ASC` | Composite | Jogos ao vivo (status IN_PROGRESS) |
| `score_events` | `gameId ASC, createdAt ASC` | Composite | Play-by-play ordenado |
| `standings` | `categoryId ASC, points DESC, wins DESC` | Composite | Leaderboard |
| `checkins` | `gameId ASC, status ASC` | Composite | Check-ins por jogo |
| `users` | `role ASC, active ASC` | Composite | Listagem por role (admin) |

**Nota:** Índices são criados via `firestore.indexes.json` no deploy ou via console. Single-field indexes são automáticos.

---

## Estratégia de Error Handling e Retry

### Camada 1: HTTP Trigger (syncEntity)

- **Idempotência:** campo `version` impede escritas antigas sobrescreverem dados novos
- **Retry:** Backend Spring Boot implementa retry exponencial (3 tentativas: 1s, 4s, 16s) com jitter
- **Timeout:** 60s na Cloud Function; backend timeout 5s (fire-and-forget após tentativas)
- **Falha definitiva:** log estruturado + alerta no Cloud Monitoring → reconciler corrige em ≤15min

```java
// Spring Boot - retry config
@Retryable(
    value = {RestClientException.class, SocketTimeoutException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 1000, multiplier = 4, maxDelay = 16000)
)
public void dispatch(SyncEvent event) { ... }
```

### Camada 2: Firestore onWrite (firestoreMirror)

- **Não escreve em PostgreSQL** (read-only contra PG) — evita loops infinitos
- **Marca com `_syncedBy: "firestoreMirror"`** para o próximo trigger não reprocessar
- **Falha silenciosa:** log warn + métrica `firestore_mirror_failures_total`; reconciler cobre

### Camada 3: Reconciliação (reconcileFirestore)

- **Idempotente por construção** (lê `updated_at` desde última execução)
- **Lock distribuído** via Firestore doc `locks/reconcile-{collection}` (TTL 14min)
- **Falha:** alerta se > 2 execuções consecutivas falharem; re-tenta na próxima janela (15min)

### Dead Letter Queue (DLQ)

Para falhas persistentes (> 1h sem sync bem-sucedido):

```
Firestore collection: `sync_failures/{id}`
{
  collection: string,
  entityId: string,
  lastError: string,
  attempts: number,
  firstFailedAt: Timestamp,
  lastAttemptAt: Timestamp,
  payload: Map
}
```

- **Trigger:** Cloud Function `onSyncFailureCreated` (Firestore trigger) envia alerta ao Slack/email
- **Resolução manual:** Admin Web tem tela "Sync Failures" para reprocessar ou descartar

---

## Estratégia de Reconciliação

### Reconciliação Incremental (a cada 15min)

```sql
-- Query executada pelo reconcileFirestore
SELECT id, * FROM teams
WHERE updated_at > $1
  AND deleted_at IS NULL
ORDER BY updated_at ASC
LIMIT 500;
```

- Janela: últimos 15min
- Batch: até 500 entidades por iteração
- Lock por collection (evita execução paralela)

### Reconciliação Full (diária, 03:00 BRT)

- Recarrega **todas** as entidades ativas
- Detecta drift acumulado (entidades que saíram do espelho por falha de sync)
- Detecta fantasmas (entidades em Firestore que não existem mais em PG)
- Marca fantasmas com `deleted: true` + `deletedAt: now()`

### Purge de Deletados (semanal, domingo 04:00 BRT)

- Remove de Firestore docs com `deletedAt < now() - 90d`
- Compliance LGPD + economia de storage

---

## Monitoramento

### Métricas Cloud Monitoring

| Métrica | Tipo | Descrição | Alerta |
|---------|------|-----------|--------|
| `sync_entity_latency_ms` | Histogram | Latência end-to-end (backend → Firestore) | p95 > 2000ms por 5min |
| `sync_entity_success_total` | Counter | Syncs bem-sucedidos por collection | — |
| `sync_entity_failure_total` | Counter | Syncs com falha por collection/error | > 10/min |
| `firestore_mirror_triggers_total` | Counter | Triggers do safety net | > 5% do total de writes |
| `reconcile_drift_detected` | Counter | Divergências encontradas pelo reconciler | > 0 em 3 janelas consecutivas |
| `sync_failures_dlq_size` | Gauge | Tamanho da DLQ | > 0 |

### Logs Estruturados

Todos os Cloud Functions emitem JSON estruturado com:

```json
{
  "severity": "INFO|WARN|ERROR",
  "message": "Synced teams/abc-123 v5 op=UPDATE",
  "collection": "teams",
  "entityId": "abc-123",
  "operation": "UPDATE",
  "version": 5,
  "triggerSource": "syncEntity|firestoreMirror|reconcileFirestore",
  "latencyMs": 234,
  "userId": "backend-service",
  "traceId": "..."
}
```

### Dashboards Grafana / Cloud Console

1. **Sync Health** — Latência p50/p95/p99, success rate, failures por collection
2. **Drift Detection** — Quantidade de divergências corrigidas pelo reconciler (deve ser ~0)
3. **DLQ** — Tamanho e aging dos itens na dead letter queue
4. **Cost** — Invocations, GB-seconds, egress

### Alertas (Cloud Monitoring → Slack/Email)

| Condição | Severidade | Canal |
|----------|-----------|-------|
| sync_entity_failure_total > 10/min por 5min | P3 | Slack `#ops-alerts` |
| reconcile_drift_detected > 0 por 3 ciclos | P2 | Slack `#ops-alerts` + email |
| sync_failures_dlq_size > 0 | P2 | Slack `#ops-alerts` + email |
| sync_entity_latency p95 > 5000ms por 10min | P2 | Slack `#ops-alerts` |
| firestore_mirror_triggers > 20% do total (sinal de problema) | P3 | Slack `#ops-alerts` |

---

## Segurança

### Autenticação Service-to-Service

- **Backend → Cloud Function:** header `X-Service-Key` validado contra Secret Manager (`INTERNAL_API_KEY`)
- **Rotação:** chave rotacionada a cada 90 dias via Secret Manager
- **mTLS (opcional futuro):** Cloud SQL Auth Proxy + IAM para PostgreSQL

### Conexão PostgreSQL

- **Cloud SQL Connector** dentro das Cloud Functions (não expõe IP público)
- **Credenciais:** Secret Manager (`PG_CONNECTION_STRING`)
- **Princípio de menor privilégio:** usuário `cqrs_reader` com SELECT apenas nas tabelas espelhadas

### Firestore Security Rules

```
match /teams/{teamId} {
  allow read: if request.auth != null;
  allow write: if false; // Apenas Cloud Functions (admin SDK) escreve
}
match /sync_failures/{docId} {
  allow read, write: if request.auth.token.role == 'SUPER_ADMIN';
}
```

- Apps leem via SDKs com `request.auth != null`
- Escritas **somente** pelo Admin SDK (Cloud Functions com service account)
- Cloud Function service account com IAM `roles/datastore.user`

### Dados Sensíveis

**NUNCA** são espelhados para Firestore:
- `users.password_hash`
- `password_reset_tokens.token`
- `users.email` (apenas `userId` para referência de audit logs)
- Tokens de integração (Stripe, etc.)

Firestore é público para leitura de apps autenticados → não pode conter PII sensível.

---

## Plano de Implementação

### Sprint 1 (setup base)
- [ ] Criar projeto Firebase + ativar Firestore + Cloud Functions Gen 2
- [ ] Configurar Secret Manager: `INTERNAL_API_KEY`, `PG_CONNECTION_STRING`
- [ ] Service account `cqrs-sync@` com permissões Firestore
- [ ] Implementar `syncEntity` (HTTP trigger) com auth
- [ ] Deploy inicial em ambiente de dev

### Sprint 2 (entidades core)
- [ ] Mappers para: Organization, Competition, Category, Team
- [ ] Backend: `CqrsSyncService` + `SyncEvent` + hook `@TransactionalEventListener`
- [ ] Testes de integração: backend → Cloud Function → Firestore
- [ ] Índices Firestore configurados (`firestore.indexes.json`)

### Sprint 3 (entidades restantes + safety net)
- [ ] Mappers para: Athlete, RosterEntry, Round, Game, ScoreEvent, Standing, CheckIn
- [ ] Implementar `firestoreMirror` (onWrite safety net)
- [ ] Testes de drift: simular falha e validar correção automática
- [ ] Security Rules no Firestore

### Sprint 4 (reconciliação + monitoramento)
- [ ] Implementar `reconcileFirestore` (15min) + `reconcileFull` (diário)
- [ ] DLQ + alertas no Cloud Monitoring
- [ ] Dashboard Grafana/Cloud Console
- [ ] Runbook para incidentes de sync

### Sprint 5 (hardening)
- [ ] Load test: 1000 writes/min durante 30min
- [ ] Chaos test: derrubar Cloud Function e validar reconciler
- [ ] Teste de segurança: pentest nas Cloud Functions e Firestore Rules
- [ ] Documentação operacional (runbooks)

---

## Critérios de Aceitação

- [ ] Sincronização PostgreSQL → Firestore em < 2s p95 (HTTP trigger)
- [ ] Reconciliação detecta e corrige drift em ≤ 15min
- [ ] Zero dados sensíveis em Firestore (validado por scan automático)
- [ ] DLQ alimentada em caso de falha persistente + alerta disparado
- [ ] Cloud Functions deployadas em `southamerica-east1` (latência BR)
- [ ] Custos < $20/mês no Free Tier + Blaze pay-as-you-go para picos
- [ ] Runbook publicado para SRE/on-call
- [ ] Testes E2E cobrindo fluxo write→sync→read em CI

---

## Riscos e Mitigações

| Risco | Probabilidade | Impacto | Mitigação |
|-------|---------------|---------|-----------|
| Drift silencioso (sync falha + reconciler falha) | Média | Alto | DLQ + alertas P2 + `reconcileFull` diário como rede extra |
| Custos de Firestore explodirem (leituras excessivas) | Média | Médio | Cache no client (Flutter), Security Rules limitando `list` operations, budget alerts |
| Loop infinito (Firestore onWrite → backend → onWrite) | Baixa | Alto | Flag `_syncedBy` + `firestoreMirror` é read-only contra PG |
| Vendor lock-in (Firebase) | Média | Médio | Camada de mappers isolada; PostgreSQL permanece source of truth → possível migração para Pub/Sub futuro |
| Latência Cloud SQL → Cloud Function | Baixa | Médio | Cloud SQL Connector (não TCP/IP público), região `southamerica-east1` |
| Cold start de Cloud Functions | Alta | Médio | Min instances = 1 para `syncEntity`; aceitável p99 para safety net |
| Falha em horário de pico (jogo ao vivo) | Média | Alto | Reconciliação de 15min é aceitável; alert P2 se drift > 0 durante jogos |

---

## Alternativas Consideradas

### A. PostgreSQL CDC (Debezium → Pub/Sub → Cloud Function)
- **Prós:** Captura 100% das mudanças, baixo acoplamento
- **Contras:** Infraestrutura extra (Debezium, Kafka/Pub/Sub), complexidade operacional, custo
- **Decisão:** Volume de 10-20 TPS não justifica; HTTP trigger é suficiente

### B. Firestore SDK direto no Backend (dual-write)
- **Prós:** Simples, sem Cloud Function
- **Contras:** Sem retry, sem reconciliação, falha = drift imediato, lock-in no backend
- **Decisão:** Cloud Function desacopla e permite evolução independente

### C. Event Broker dedicado (Kafka/Pub/Sub)
- **Prós:** Escalável, replay, padrão event sourcing
- **Contras:** Custo + ops de broker, over-engineering para volume atual
- **Decisão:** Reservar como evolução futura se TPS > 100

### D. Apenas Cloud Scheduler (sem HTTP trigger)
- **Prós:** Zero acoplamento backend
- **Contras:** Latência de até 15min para qualquer leitura (inaceitável para live scores)
- **Decisão:** Scheduler apenas como reconciliador, não como caminho primário

---

## Referências

- [ADR-001](ADR-001-nova-filosofia-arquitetura.md) – Filosofia de arquitetura
- [ADR-002](ADR-002-postgres-firestore-cqs.md) – PostgreSQL + Firestore CQS Light
- [ADR-003](ADR-003-modular-monolith.md) – Modular Monolith
- [ADR-004](ADR-004-firebase-auth-migration.md) – Migração Firebase Auth
- [ADR-005](ADR-005-staging-efemero-e2e.md) – Staging efêmero E2E
- [Cloud Functions Gen 2 docs](https://cloud.google.com/functions/docs/2nd-gen/overview)
- [Firestore Security Rules](https://firebase.google.com/docs/firestore/security/get-started)
- [Cloud SQL Connector](https://cloud.google.com/sql/docs/postgres/connect-overview)

---

**Status:** Proposto
**Versão:** 1.0
**Próxima revisão:** Após Sprint 5 (hardening) ou ao atingir 50 TPS sustentados
