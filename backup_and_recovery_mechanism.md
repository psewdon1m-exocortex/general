# Механизм резервного сохранения и восстановления Exocortex

Документ описывает фактически реализованный поток recovery-архивов между
Settings проекта, локальным Neptune и Saturn, а также обратный поток
восстановления проекта из сохранённого ZIP.

Состояние реализации и схемы сверены по исходникам на 2026-09-20. Нормативные
требования к составу архива, безопасности ZIP и restore находятся в
[Part 03. Backup And Recovery](./PART_03_BACKUP_AND_RECOVERY.md).

## 1. Что изменилось относительно старой схемы

- Расписание редактируется в Settings самого проекта, но его authoritative
  revision хранится в Saturn. Neptune является аутентифицированным relay и
  исполнителем политики.
- Актуальные локальные маршруты — `/policy` и `/policy/runs`. Старые mutation
  routes `/schedule` и `/runs` возвращают HTTP 426.
- Запись policy использует `requestId` и `expectedRevision`, поэтому повтор
  запроса идемпотентен, а конкурентное изменение возвращает conflict.
- Ручной `Back up to Saturn now` не запускает worker напрямую. Команда
  сохраняется в Saturn, затем Neptune получает её через outbound check-in.
- Registry Neptune расположен в `/var/lib/neptune/projects.json`, а не в
  `/etc/neptune/projects.json`.
- Файлы credentials проекта находятся в `/etc/neptune/clients/`.
- Текущая реализация Neptune передаёт Saturn chunks размером не более 1 МиБ,
  даже если локальный default и capabilities допускают больше.
- Финальный путь Saturn включает namespace и, для не-default установки,
  `deploymentId`.
- Committed backup публикуется в каталоге Saturn Files. Оператор скачивает его
  оттуда и восстанавливает через собственный restore workflow проекта.
- Restore сохраняет backup policy как pending intent. Автоматические запуски
  возобновляются только после проверки source, destination, enrollment и
  применения новой revision.

## 2. Предварительное подключение проекта

До настройки расписания проект должен быть enrolled в Saturn и зарегистрирован
в единственном Neptune daemon данного Linux-хоста.

```text
Saturn → Backup/Synchronization enrollment
 │
 │ оператор создаёт одноразовый setup code
 │ срок действия: 15 минут
 ▼
Settings проекта → Initialize / Repair Neptune
 │
 │ передаёт setup code локальному Updater
 ▼
Updater Linux
 │
 ├─ устанавливает или повторно использует общий neptuned
 ├─ redeem-ит setup code в Saturn
 ├─ получает scoped producer identity:
 │    serviceId
 │    slug
 │    namespaceSlug
 │    deploymentId
 │    Saturn producer token
 │
 ├─ создаёт отдельные credentials проекта:
 │    /etc/neptune/clients/<project>.control.token
 │    /etc/neptune/clients/<project>.export.token
 │    /etc/neptune/clients/<project>.saturn.token
 │
 ├─ создаёт registration env:
 │    /etc/neptune/projects/<project>.env
 │
 └─ выполняет:
      neptuned register-project <project-id> <project-env-file>
 ▼
Neptune project registry
 │
 │ /var/lib/neptune/projects.json
 │
 │ хранит для каждого deployment:
 │ project ID
 │ loopback export URL
 │ пути к control/export/Saturn token files
 │ Saturn slug или Register key для него
 │ enabled/interval/NextRunAt
 │ applied policy revision и paused flag
 │ optional mirror/reader registration
 ▼
Проект
 │
 │ получает bind-mounted control/export tokens
 │ и после enrollment пересоздаётся только его Compose service,
 │ чтобы новый credential inode гарантированно попал в контейнер
```

Один `neptuned` обслуживает несколько проектов на хосте. Проекты имеют разные
control, export и Saturn producer credentials и не могут управлять чужой
политикой или записывать архивы в чужой namespace.

## 3. Настройка расписания и ручной запуск

```text
Settings проекта
 │
 │ пользователь:
 │
 ├─ включает Enable automatic backups
 ├─ задаёт interval в целых часах, от 1 до 8760
 └─ либо нажимает Back up to Saturn now
 ▼
Frontend проекта
 │
 ├─ GET <project-policy-api>
 ├─ PUT <project-policy-api>
 ├─ GET <project-policy-api>/runs
 └─ POST <project-policy-api>/runs
 │
 │ сохраняет requestId только как hint для безопасного retry
 │ и каждые 5 секунд обновляет policy/run status
 ▼
Backend проекта
 │
 │ проверяет operator session и mutation/CSRF authorization
 │
 │ обращается к локальному Neptune через Unix socket:
 │ /run/neptune/neptuned.sock
 │
 │ Host: neptune.local
 │ X-Neptune-Token: <PROJECT_CONTROL_TOKEN>
 │
 ├─ GET /v1/projects/<deployment-id>/policy
 │
 ├─ PUT /v1/projects/<deployment-id>/policy
 │    {
 │      "kind":"schedule",
 │      "pipeline":"archive",
 │      "enabled":true,
 │      "intervalHours":24,
 │      "expectedRevision":17,
 │      "requestId":"<uuid>"
 │    }
 │
 ├─ POST /v1/projects/<deployment-id>/policy/runs
 │    {
 │      "pipeline":"archive",
 │      "requestId":"<uuid>"
 │    }
 │
 ├─ GET /v1/projects/<deployment-id>/policy/runs
 │
 └─ GET /v1/projects/<deployment-id>/status
 ▼
Neptune Linux
 │
 ├─ находит точную registration по deployment-id
 ├─ читает соответствующий control token file
 ├─ сравнивает X-Neptune-Token в constant time
 ├─ не принимает service ID, destination или credential из request body
 └─ передаёт policy operation в Saturn от имени enrolled producer
 ▼
Kernel Register
 │
 │ Authorization: Bearer <KERNEL_SERVICE_TOKEN>
 │
 │ Neptune разрешает только нужные координаты Saturn:
 │ services.saturn.sni
 │ services.saturn.port
 │
 │ исходный Register snapshot содержит только volt:// references
 │ snapshot с checksum сохраняется как:
 │ /var/lib/neptune/register-lkg.json
 │
 │ resolved значения возвращаются Kernel и живут только в памяти
 ▼
Saturn service-owned policy API
 │
 │ Authorization: Bearer <PROJECT_SATURN_PRODUCER_TOKEN>
 │
 ├─ GET  /api/v1/neptune/agent/policy
 ├─ PUT  /api/v1/neptune/agent/policy
 ├─ GET  /api/v1/neptune/agent/policy/runs
 └─ POST /api/v1/neptune/agent/policy/runs
 ▼
Saturn policy database
 │
 ├─ producer identity выбирает service/deployment scope
 ├─ expectedRevision защищает от lost update
 ├─ requestId обеспечивает idempotency
 ├─ schedule mutation увеличивает desired revision
 ├─ изменение interval не создаёт ручной run
 └─ manual run создаёт command:
      id = requestId
      kind = archive.run
      state = pending
```

Ответ policy имеет форму:

```json
{
  "schema": "exocortex.backup.policy.v1",
  "owner": "service",
  "scope": {
    "serviceId": "...",
    "service": "chronos",
    "deploymentId": "vps-a",
    "mirrorRoot": null
  },
  "revision": 18,
  "appliedRevision": 17,
  "paused": false,
  "archive": {
    "enabled": true,
    "intervalHours": 24
  },
  "mirror": null,
  "observed": {
    "online": true,
    "lastSeenAt": "...",
    "version": "...",
    "archive": {
      "state": "complete",
      "lastSuccessAt": "...",
      "nextRunAt": "..."
    }
  }
}
```

`revision` означает authoritative desired state в Saturn. `appliedRevision`
означает revision, которую уже подтвердил Neptune. Сохранённое расписание не
равно доказанному backup: UI отдельно показывает last seen, last success,
NextRunAt, active state и latest error.

## 4. Доставка policy и команд в Neptune

```text
Neptune RemoteControlWorker
 │
 │ каждые 15 секунд выполняет outbound check-in
 │ входящий порт на VPS не нужен
 ▼
POST /api/v1/neptune/agent/check-in
Authorization: Bearer <PROJECT_SATURN_PRODUCER_TOKEN>
 │
 │ {
 │   "clientInstanceId":"<stable-client-id>",
 │   "projectId":"<deployment-id>",
 │   "version":"<neptune-version>",
 │   "appliedRevision":17,
 │   "archive":{
 │     "active":false,
 │     "state":"complete",
 │     "lastAttemptAt":"...",
 │     "lastSuccessAt":"...",
 │     "nextRunAt":"...",
 │     "error":null
 │   },
 │   "commandResults":[...]
 │ }
 ▼
Saturn
 │
 ├─ записывает observed state и lastSeenAt
 ├─ подтверждает завершённые command results
 ├─ возвращает desired policy revision
 └─ возвращает до 20 pending commands с expiresAt
 ▼
Neptune
 │
 ├─ атомарно применяет более новую revision к projects.json
 ├─ сохраняет applied revision
 ├─ сохраняет remote command journal в SQLite
 └─ выполняет archive.run только один раз для данного command ID
```

Если Saturn временно недоступен, последнее уже применённое локальное расписание
остаётся активным. Новая policy считается применённой только после check-in или
успешного синхронного relay и записи той же revision в registry.

## 5. Создание recovery ZIP

```text
Neptune BackupWorker
 │
 ├─ каждые 30 секунд проверяет локальный NextRunAt
 ├─ либо принимает полученный archive.run command
 ├─ не запускает работу при policy_paused = true
 ├─ допускает только один активный archive run одного проекта
 └─ ограничивает общую параллельность значением MaxParallelProjects
      default: 4
 ▼
Внутренний endpoint проекта
 │
 │ зарегистрированный loopback URL, обычно:
 │ POST http://127.0.0.1:<port>/api/.../internal/neptune/backup
 │
 │ Authorization: Bearer <PROJECT_EXPORT_TOKEN>
 │ Accept-Encoding: identity
 │
 │ endpoint недоступен через публичный reverse proxy
 ▼
Общий backup builder проекта
 │
 ├─ создаёт тот же полный logical ZIP,
 │    который используется ручным backup и restore
 ├─ Content-Type: application/zip
 ├─ Content-Length: точный размер
 ├─ не использует content encoding
 ├─ включает non-secret backup policy intent
 └─ не включает bearer tokens, setup codes, .env и plaintext secrets
 ▼
Neptune ProjectBackupExporter
 │
 ├─ принимает только loopback HTTP(S)
 ├─ ограничивает архив максимумом 8 ГиБ
 ├─ ограничивает полный export одним часом
 ├─ требует progress не реже одного раза в 60 секунд
 └─ потоково записывает ответ без преобразования ZIP
 ▼
Neptune spool
 │
 ├─ создаёт файл mode 0600:
 │    /var/lib/neptune/spool/<project-id>-<run-id>.zip
 ├─ записывает точные bytes проекта
 ├─ Flush(flushToDisk: true)
 ├─ вычисляет SHA-256 во время записи
 ├─ сверяет Content-Length
 └─ сохраняет run metadata в SQLite WAL:
      /var/lib/neptune/neptune.db
```

Neptune не открывает ZIP, не добавляет и не удаляет members, не перепаковывает,
не сжимает повторно и не меняет его формат. Saturn получает ровно те bytes,
которые принимает ручной restore проекта.

Локальные состояния archive run:

```text
exporting
  → spooled
  → uploading
  → complete

ошибка после создания spool:
  → retry-wait

неполный export:
  → export-failed
```

## 6. Разрешение destination

```text
Neptune Linux
 │
 │ читает /etc/neptune/kernel.token
 │
 └─ GET /api/v1/register/snapshot
      Authorization: Bearer <KERNEL_SERVICE_TOKEN>
 ▼
Kernel Register
 │
 │ Neptune запрашивает только:
 │ services.saturn.sni
 │ services.saturn.port
 │ services.saturn.paths.backup_ingest
 │ и, если slug не закреплён registration:
 │ services.<project>.backup.saturn_slug
 │
 │ POST /api/v1/register/resolve разрешает volt:// references
 │ реальные значения остаются в памяти Neptune
 ▼
Neptune Linux
 │
 │ отдельно читает локальный producer token:
 │ /etc/neptune/clients/<project-id>.saturn.token
 │
 └─ строит HTTPS base URL Saturn Backup API
```

Изменение Register применяется только к новой работе и не перенаправляет уже
идущую upload-сессию. В рамках процесса Neptune может продолжить с последним
успешно разрешённым Saturn target; после полного рестарта resolved secrets
должны быть снова получены через Kernel.

## 7. Resumable upload в Saturn

```text
Saturn Backup API
 │
 ├─ GET /api/v1/backups/capabilities
 │    └─ Neptune требует:
 │       schema = saturn.backup-ingest.capabilities.v1
 │       protocolVersion = 1
 │       resumable = true
 │       checksum = sha256
 │       maxChunkBytes >= 1
 │
 ├─ POST /api/v1/backups/<service-slug>/runs
 │    Authorization: Bearer <PROJECT_SATURN_PRODUCER_TOKEN>
 │    Idempotency-Key: neptune-<sha256(client+project+run)>
 │
 │    {
 │      "filename":"<project>-<yyyyMMddTHHmmssZ>.zip",
 │      "createdAt":"<UTC ISO timestamp>",
 │      "backupType":"full",
 │      "expectedSize":...,
 │      "sha256":"...",
 │      "sourceVersion":"<neptune-version>",
 │      "encrypted":false
 │    }
 │
 ├─ HEAD /api/v1/backups/<service-slug>/runs/<saturn-run-id>/upload
 │    Authorization: Bearer <PROJECT_SATURN_PRODUCER_TOKEN>
 │    └─ Saturn возвращает текущий Upload-Offset
 │
 ├─ PATCH /api/v1/backups/<service-slug>/runs/<saturn-run-id>/upload
 │    Authorization: Bearer <PROJECT_SATURN_PRODUCER_TOKEN>
 │    Upload-Offset: <exact-offset>
 │    Content-Length: <exact-chunk-size>
 │    Content-Type: application/offset+octet-stream
 │    └─ текущий Neptune отправляет chunks не более 1 МиБ
 │
 └─ POST /api/v1/backups/<service-slug>/runs/<saturn-run-id>/complete
      Authorization: Bearer <PROJECT_SATURN_PRODUCER_TOKEN>
      └─ server-side verification может выполняться до 15 минут
```

Saturn авторизует namespace не по одному path slug, а по producer identity.
Path slug должен совпасть с этой identity. Producer может создавать, продолжать,
завершать и читать статус только собственных runs; он не может перечислять,
скачивать или удалять committed backups и менять retention.

## 8. Commit и receipt в Saturn

```text
Saturn Backup Ingest
 │
 ├─ резервирует per-service quotas транзакционно
 ├─ проверяет maxBackupBytes
 ├─ проверяет dailyQuotaBytes и storedQuotaBytes
 ├─ проверяет maxConcurrentRuns
 ├─ сохраняет partial object под:
 │    _system/incoming/backups/<service-id>/<run-id>.part
 ├─ принимает только exact Upload-Offset
 └─ после complete повторно вычисляет полный size и SHA-256
 ▼
Проверка результата
 │
 ├─ size или SHA-256 не совпали
 │    ├─ partial object удаляется или помечается failed
 │    ├─ final object не публикуется
 │    └─ receipt не создаётся
 │
 └─ size и SHA-256 совпали
      ├─ partial object атомарно переносится в final path
      ├─ previous backup не перезаписывается
      ├─ run становится complete
      └─ создаётся immutable receipt
 ▼
Финальное хранение
 │
 │ backups/<namespace-slug>/<deployment-id?>/<YYYY>/<MM>/<DD>/
 │   <ISO-stamp>_full_<saturn-run-id>_<filename>.zip
 │
 │ deployment-id отсутствует только для legacy/default identity
 │
 │ пример:
 │ backups/chronos/vps-a/2026/09/20/
 │   2026-09-20T12-00-00-000Z_full_<run-id>_chronos-20260920T120000Z.zip
 ▼
Saturn receipt
 │
 │ {
 │   "schema":"vault.service-backup-receipt.v1",
 │   "runId":"<saturn-run-id>",
 │   "serviceId":"<producer-service-id>",
 │   "serviceSlug":"<service-slug>",
 │   "logicalPath":"/backups/chronos/vps-a/...zip",
 │   "sizeBytes":...,
 │   "sha256":"...",
 │   "committedAt":"..."
 │ }
 ▼
Saturn Files catalog
 │
 ├─ принимает уже существующий verified object
 ├─ создаёт folder/resource metadata под системным Backups root
 ├─ сохраняет тот же size, SHA-256 и MIME type
 └─ делает архив доступным owner через обычный Files UI
```

Retention выбирается Saturn по policy producer identity. Reference defaults:
7 daily, 4 weekly, 12 monthly и 3 yearly. Neptune не имеет права удалять
удалённые архивы.

## 9. Завершение и retry в Neptune

```text
Neptune Linux
 │
 │ сравнивает Saturn receipt с локальным spool:
 │ receipt.sizeBytes == local size
 │ receipt.sha256 == local SHA-256
 │
 ├─ совпадает
 │    ├─ сохраняет state = complete в SQLite
 │    ├─ очищает SpoolPath в run metadata
 │    ├─ удаляет локальный ZIP
 │    ├─ для scheduled run ставит следующий NextRunAt через N часов
 │    ├─ manual run не сдвигает независимое расписание
 │    └─ передаёт success в следующем Saturn check-in
 │
 └─ network/restart/server/receipt error
      ├─ оставляет ZIP в spool
      ├─ сохраняет metadata и error в SQLite
      ├─ переводит run в retry-wait
      ├─ повторяет попытку через 15 минут
      ├─ повторно использует idempotency key
      └─ продолжает с Upload-Offset, подтверждённого Saturn
```

При старте Neptune читает SQLite и возобновляет recoverable runs, если их spool
существует. Disabling schedule запрещает новые scheduled runs, но не уничтожает
уже принятый upload. Orphan spool очищается только по journal state и bounded
startup cleanup.

## 10. Ручной backup проекта

Ручной download не проходит через Neptune и Saturn:

```text
Settings проекта → Create and download snapshot
 │
 ▼
Backend проекта → общий backup builder
 │
 ▼
тот же logical ZIP
 │
 ▼
браузер пользователя
```

Manual, automatic и pre-update backup используют один формат и один набор
authoritative данных. Отличается только transport destination:

```text
Manual:    builder → browser
Automatic: builder → Neptune spool → Saturn
Update:    builder → browser save gate → Updater RAM/tmpfs
Restore:   любой совместимый ZIP → project inspect/restore workflow
```

## 11. Восстановление проекта из Saturn backup

Neptune и Saturn не интерпретируют доменные records проекта. Restore выполняет
сам проект, потому что только его версия знает schema, migrations, write barrier
и порядок восстановления.

```text
Saturn owner UI → Files → Backups
 │
 │ оператор выбирает архив конкретного service/deployment/date
 ▼
Saturn Files API
 │
 └─ GET /api/v1/files/<resource-id>/content
      Cookie/session owner authorization
      recent reauthentication для защищённого ресурса
      Range поддерживается
      ETag: "sha256-<artifact-sha256>"
 ▼
Компьютер оператора
 │
 │ сохраняет точный ZIP
 │ для encrypted profile отдельно сохраняется recovery key/identity
 ▼
Settings проекта → Restore snapshot
 │
 │ пользователь выбирает локальный ZIP через native file picker
 ▼
Project preflight / inspect
 │
 ├─ проверяет compressed-size limit
 ├─ проверяет ZIP central directory
 ├─ отклоняет absolute paths, .., links и duplicate names
 ├─ проверяет member count, member sizes и expansion ratio
 ├─ проверяет manifest schema и source compatibility
 ├─ проверяет allow-listed members
 ├─ проверяет SHA-256 каждого declared member
 ├─ парсит и валидирует records до mutation
 └─ показывает оператору filename, createdAt, schema, scope и replace mode
 ▼
Явное подтверждение restore
 │
 │ пользователь подтверждает замену текущего состояния
 ▼
Project restore engine
 │
 ├─ создаёт transaction rollback или verified pre-restore snapshot
 ├─ включает maintenance/write barrier
 ├─ восстанавливает settings и identities
 ├─ восстанавливает основные records и relations
 ├─ атомарно переключает file state, если он есть
 ├─ перестраивает derived indexes
 ├─ завершает старые operator sessions
 ├─ выполняет invariants и health checks
 │
 ├─ success
 │    ├─ commit
 │    ├─ audit с archive digest и result
 │    └─ повторная авторизация оператора
 │
 └─ failure
      ├─ rollback database transaction
      ├─ либо возврат verified pre-restore snapshot
      └─ сервис не сообщает healthy partial restore
```

### Application restore routes

| Проект | Inspect/preflight | Apply restore |
| --- | --- | --- |
| Kernel | `POST /api/backup/inspect` | `POST /api/backup/restore` |
| Volt | `POST /api/v1/backup/inspect` | `POST /api/v1/backup/restore` |
| Chronos | `POST /api/backup/inspect` | `POST /api/backup/restore` |
| Laboratory | `POST /api/admin/restore/inspect` | `POST /api/admin/restore` |
| Perimetr | `POST /v1/backups/preflight` | `POST /v1/backups/import` |
| Mastermind | owner operation `kind=restore`, затем inspect | apply внутри durable restore operation |
| Saturn | `/api/v1/operator/recovery/restores/*` | validate, затем explicit apply |

Имена маршрутов различаются, но boundary один: полная проверка архива должна
завершиться до первого изменения live state.

## 12. Восстановление backup policy

Backup ZIP содержит non-secret intent расписания, но восстановление данных не
должно автоматически запускать старое расписание или повторять старые manual
commands.

```text
Project restore engine
 │
 ├─ извлекает из ZIP:
 │    schema = exocortex.backup.intent.v1
 │    archive.enabled
 │    archive.intervalHours
 │    optional mirror.enabled/intervalMinutes
 │    sourceRevision
 │
 ├─ сохраняет intent как local pending restore journal
 └─ показывает:
      Restored policy · pending verification
      Execution is paused
 ▼
Settings → Verify and resume restored policy
 │
 ▼
Backend проекта → Neptune
 │
 ├─ GET /v1/projects/<deployment>/policy
 │
 └─ PUT /v1/projects/<deployment>/policy
      {
        "kind":"restore",
        "requestId":"<stable-uuid>",
        "expectedRevision":<current-revision>,
        "archive":{
          "enabled":true,
          "intervalHours":24
        },
        "mirror":null
      }
 ▼
Saturn
 │
 ├─ проверяет expectedRevision
 ├─ сохраняет восстановленный desired intent
 ├─ устанавливает policy_paused = true
 ├─ увеличивает revision
 └─ завершает старые pending archive.run/mirror.run как
      policy_restored_pending_verification
 ▼
Neptune verification
 │
 ├─ HEAD <registered-project-export-url>
 │    Authorization: Bearer <PROJECT_EXPORT_TOKEN>
 │    требует X-Neptune-Ready: 1
 │
 ├─ для mirror также проверяет source endpoint
 ├─ проверяет enrolled Saturn destination
 └─ при любой ошибке оставляет policy paused и journal сохранённым
 ▼
Backend проекта → Neptune → Saturn
 │
 └─ PUT policy
      {
        "kind":"resume",
        "requestId":"<stable-uuid>",
        "expectedRevision":<restored-revision>
      }
 ▼
Saturn + Neptune
 │
 ├─ снимают paused
 ├─ создают следующую revision
 ├─ Neptune атомарно применяет её локально
 ├─ NextRunAt рассчитывается заново по восстановленному interval
 └─ pending journal удаляется только когда
      revision == appliedRevision и paused == false
```

Недоступный Kernel, Saturn, export endpoint или mirror destination не приводит
к подмене интервала на default и не включает расписание частично.

## 13. Особая ветка Mastermind

Mastermind создаёт зашифрованный полный archive и связывает его с generation
writer barrier.

```text
Neptune → Mastermind Core
 │
 │ POST /api/internal/neptune/backup
 │ Authorization: Bearer <PROJECT_EXPORT_TOKEN>
 │ X-Neptune-Purpose: archive
 ▼
Mastermind Core
 │
 ├─ проверяет, что restored backup policy не pending
 ├─ останавливает/координирует writers
 ├─ создаёт consistent generation snapshot
 ├─ шифрует recovery ZIP
 └─ возвращает:
      Content-Type: application/zip
      Content-Length: <size>
      X-Mastermind-Generation: <generation>
      X-Content-SHA256: <sha256>
 ▼
Neptune
 │
 ├─ сверяет generation receipt с фактически записанным spool
 ├─ отправляет Saturn run с encrypted = true
 └─ требует capability archiveEncryptionDeclaredPerRun = true
 ▼
Saturn commit
 │
 │ size/SHA-256 verified, receipt создан
 ▼
Neptune → Mastermind Core
 │
 └─ POST <export-url>/receipt
      Authorization: Bearer <PROJECT_EXPORT_TOKEN>
      X-Neptune-Purpose: archive
      {"generation":...,"size":...,"sha256":"..."}
 ▼
Mastermind Core
 │
 └─ подтверждает exact remote commitment этой generation
```

Восстановление Mastermind требует зашифрованный ZIP и отдельно сохранённую
recovery identity/key. Архив проверяется до mutation, затем данные и обязательное
состояние переключаются как согласованная generation; при ошибке используется
pre-restore safety backup.

## 14. Границы хранения и доверия

```text
/etc/neptune/clients/<project>.control.token
  └─ Backend проекта → локальный Neptune control API

/etc/neptune/clients/<project>.export.token
  └─ Neptune → loopback export endpoint проекта

/etc/neptune/clients/<project>.saturn.token
  └─ Neptune → Saturn policy/check-in/backup API

/var/lib/neptune/projects.json
  └─ registrations, applied policy revision, NextRunAt

/var/lib/neptune/neptune.db
  └─ backup run journal, remote commands, retry/upload metadata

/var/lib/neptune/spool/<project>-<run>.zip
  └─ временный exact-byte ZIP до verified Saturn receipt

/var/lib/neptune/register-lkg.json
  └─ checksummed Register references; resolved values не сохраняются

Saturn _system/incoming/backups/...
  └─ незавершённые resumable uploads

Saturn backups/<namespace>/<deployment>/...
  └─ immutable committed recovery archives

Компьютер оператора
  └─ скачанная recovery copy и, если требуется, отдельный recovery key
```

Основные trust boundaries:

- project control token не является export или Saturn producer token;
- каждый producer token ограничен одной service/deployment identity;
- Kernel service token разрешает только Register values, а не backup upload;
- Neptune транспортирует opaque ZIP и не отвечает за доменную корректность
  restore;
- Saturn проверяет transport integrity и quotas, но не выполняет application
  restore;
- только backend проекта валидирует schema и изменяет своё live state.

## 15. Реализация в исходниках

Основные точки входа:

- `neptune/docs/service-owned-policy.md` — актуальный policy contract;
- `neptune/docs/PROJECT_INTEGRATION.md` — регистрация проекта;
- `neptune/src/Neptune.Linux/Program.cs` — Unix-socket API Neptune;
- `neptune/src/Neptune.Linux/ServicePolicyClient.cs` — relay policy в Saturn;
- `neptune/src/Neptune.Linux/RemoteControlWorker.cs` — check-in и command queue;
- `neptune/src/Neptune.Linux/BackupWorker.cs` — scheduler, retry и upload flow;
- `neptune/src/Neptune.Core/ProjectBackupExporter.cs` — exact-byte export;
- `neptune/src/Neptune.Core/BackupCoordinator.cs` — spool и idempotency key;
- `neptune/src/Neptune.Core/SaturnBackupClient.cs` — resumable Saturn upload;
- `neptune/src/Neptune.Core/NeptuneStateStore.cs` — SQLite journal;
- `saturn/apps/api/src/service-backup-policy.ts` — authoritative policy;
- `saturn/apps/api/src/neptune-fleet.service.ts` — desired/observed state;
- `saturn/apps/api/src/backup.controller.ts` — producer and owner API;
- `saturn/packages/backup-ingest/src/backup.service.ts` — quotas, commit, receipt,
  catalog publication and retention;
- `saturn/packages/file-core/src/file.service.ts` — публикация и скачивание
  committed backup через Files;
- `kernel/src/backup-policy.js` — общий Settings policy UI;
- `kernel/server/backup-policy.js` — restore-aware policy adapter;
- `volt/server/backup-policy.js` — интеграция Volt;
- `chronos/app/backup_policy.py` — интеграция Chronos;
- `laboratory/services/api/src/backup-policy.js` — интеграция Laboratory;
- `mastermind/src/mastermind/backup_policy.py` — интеграция Mastermind.
