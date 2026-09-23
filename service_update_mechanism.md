# Механизм обновления сервисов Exocortex

Этот документ описывает фактически реализованный сквозной процесс обновления
сервисов на одном Linux-хосте: от кнопки в `Settings` до проверки новой версии,
обязательного сохранения резервной копии, установки, health-check и rollback.

Нормативные требования находятся в
[Part 05. CI, Releases And Local Updates](./PART_05_CI_RELEASES_AND_LOCAL_UPDATES.md).
Здесь собрана прикладная схема по текущим интеграциям проектов.

## 1. Участники

- **Сервис** — Kernel, Volt, Chronos, Laboratory, Saturn или Perimetr. Сервис
  показывает UI, авторизует оператора и создаёт собственный полный backup ZIP.
- **Updater** — один root-daemon на Linux-хосте. Он обслуживает несколько
  зарегистрированных сервисов, проверяет релизы и изменяет Docker Compose/systemd.
- **Kernel Register** — источник разрешённых адресов репозиториев вида
  `repositories.<component>.url`.
- **GitHub Releases** — источник квалифицированных релизов
  `<service>-v<MAJOR.MINOR.PATCH>`, manifest, подписи и deployment bundle.
- **Helper-компоненты** — общий Updater, Neptune, Gryphon и Wyvern. Они
  обновляются через тот же daemon, но без backup данных приложения.
- **Mastermind** — особый групповой сервис из Core, Runtime и Worker; для него
  используется потоковый saved-copy protocol и общий writer barrier.

## 2. Общая схема обновления сервиса

Эта ветка применяется к Kernel, Volt, Chronos, Laboratory и Saturn. Perimetr
использует тот же протокол Updater через собственные маршруты API.

```text
Settings сервиса
 │
 │ пользователь открывает Updates
 │ и нажимает Check for updates
 ▼
Frontend сервиса
 │
 ├─ POST <service-update-api>/check
 │    {"component":"<service>"}
 │
 ├─ сразу показывает Checking...
 ├─ хранит только job/request ID для восстановления UI
 └─ не получает Docker socket, release URL для установки
    или возможность выполнить произвольную команду
 ▼
Backend сервиса
 │
 │ проверяет сессию оператора и право на mutation
 │
 │ обращается к локальному Updater через Unix socket:
 │ /run/exocortex/updater.sock
 │
 │ Host: updater.local
 │ X-Updater-Token: <HEAD_CONTROL_TOKEN>
 │
 ├─ GET  /v1/health
 │       └─ требует update_protocol = 2
 │
 ├─ POST /v2/check
 │       {
 │         "head_id":"<registered-head-id>",
 │         "component":"<service>"
 │       }
 │
 ├─ GET  /v1/jobs?head_id=<registered-head-id>
 │
 └─ GET  /v1/jobs/<job-id>
 ▼
Updater Linux
 │
 │ один root-daemon обслуживает все зарегистрированные heads на хосте
 │ проверяет токен именно указанного head
 │ читает регистрацию из:
 │ /etc/exocortex/updater-heads.json
 │
 │ загружает env/profile зарегистрированного сервиса:
 │ UPDATER_SERVICE_ID
 │ UPDATER_COMPOSE_PROJECT_DIR
 │ UPDATER_COMPOSE_FILE
 │ UPDATER_COMPOSE_SERVICE
 │ UPDATER_IMAGE_VARIABLE
 │ UPDATER_VERSION_VARIABLE
 │ UPDATER_LOCAL_HEALTH_URL
 │ UPDATER_PUBLIC_HEALTH_URL
 │ UPDATER_RESTORE_URL
 │
 │ одновременно на всём хосте выполняется только одна mutation-операция
 ▼
Kernel Register
 │
 │ Updater обращается с KERNEL_SERVICE_TOKEN
 │ и получает разрешённый источник релиза:
 │ repositories.<service>.url
 │
 │ используется проверенный snapshot/last-known-good cache:
 │ /var/lib/updater/register-<head-id>.json
 ▼
GitHub Releases
 │
 │ Updater ищет stable release:
 │ <service>-v<MAJOR.MINOR.PATCH>
 │
 │ draft, prerelease, чужой prefix, downgrade
 │ и переустановка той же версии отбрасываются
 │
 └─ в Settings возвращается:
      installed_version
      available_version
      update_available
      release_url
      backup_required = true
```

Проверка обновления только показывает кандидата. Она ничего не меняет на хосте
и не считается проверкой скачанных артефактов: перед установкой Updater повторно
разрешает точный релиз и проверяет его независимо от UI.

## 3. Обязательное сохранение backup перед установкой

```text
Окно Install <version>
 │
 │ пользователь нажимает Create backup and install
 ▼
Backend сервиса
 │
 │ повторно вызывает POST /v2/check
 │ и убеждается, что выбранная версия всё ещё текущий кандидат
 │
 │ создаёт свежий полный ZIP тем же builder,
 │ который используется обычным ручным backup
 │
 │ ограничение обычного service update: 128 МиБ
 ▼
Backup receipt
 │
 │ backend вычисляет размер и SHA-256 ZIP
 │ и подписывает HMAC с помощью локального head control token
 │
 │ schema: exocortex.update-backup.v2
 │
 │ receipt связывает:
 │ request_id
 │ head_id
 │ service
 │ target version
 │ filename
 │ size
 │ SHA-256
 │ expires = текущий момент + 15 минут
 │
 └─ X-Update-Receipt: <base64url-body>.<HMAC-SHA256>
 ▼
Браузер
 │
 │ получает тот же ZIP и receipt
 │ самостоятельно сравнивает size и SHA-256 с receipt
 │
 ├─ File System Access API доступен
 │    ├─ заранее открывает showSaveFilePicker
 │    ├─ записывает ZIP
 │    ├─ дожидается close()
 │    └─ продолжает установку только после успешного сохранения
 │
 └─ обычная browser download
      ├─ инициирует скачивание ZIP
      ├─ показывает отдельный checkbox
      │  I have saved the ZIP on my computer
      └─ включает Install только после явного подтверждения
```

ZIP не пересоздаётся между сохранением и установкой. Браузер возвращает backend
те же байты, которые сохранил оператор. Ни сервис, ни Updater не оставляют
постоянную копию этого архива на диске хоста.

## 4. Передача установки в Updater

```text
Браузер
 │
 ├─ POST <service-update-api>/install/<service>
 │
 │  Content-Type: application/octet-stream
 │  X-Update-Receipt: <signed-receipt>
 │  X-Update-Saved: 1
 │  Body: <тот же ZIP>
 ▼
Backend сервиса
 │
 │ проверяет HMAC receipt, scope, expiry, size и SHA-256
 │ очищает рабочий buffer после передачи
 │
 └─ POST /v2/updates через Unix socket
      {
        "request_id":"<receipt-id>",
        "head_id":"<registered-head-id>",
        "service":"<service>",
        "version":"<exact-version>",
        "operator_saved":true,
        "backup_receipt":"<signed-receipt>",
        "backup":{
          "filename":"<original-name>.zip",
          "sha256":"<sha256>",
          "data_base64":"<same-zip>"
        }
      }
 ▼
Updater admission
 │
 ├─ повторно проверяет X-Updater-Token и head/service scope
 ├─ проверяет HMAC receipt тем же control token
 ├─ проверяет operator_saved = true
 ├─ декодирует ZIP с лимитом 128 МиБ
 ├─ повторно вычисляет SHA-256
 ├─ проверяет request_id/idempotency
 ├─ повторный идентичный request возвращает существующий job
 ├─ тот же request_id с другим target/payload отклоняется
 └─ захватывает host-wide mutation lock
 ▼
Durable job
 │
 │ Updater сразу возвращает 202 и job ID
 │
 │ metadata:
 │ /var/lib/updater/jobs/<job-id>.json
 │
 │ backup ZIP остаётся только в памяти процесса
 │
 └─ UI опрашивает GET /v1/jobs/<job-id>
      обычно каждые 1,5 секунды, с reconnect после ошибок
```

Основные состояния job:

```text
REQUESTED
  → BACKUP_VERIFIED
  → ARTIFACT_VERIFIED
  → PULLING
  → APPLYING
  → HEALTH_CHECK
  → COMPLETED

При ошибке после начала mutation:
  → ROLLING_BACK
  → ROLLED_BACK | ROLLBACK_FAILED

При ошибке до mutation:
  → FAILED
```

## 5. Проверка релиза и установка

```text
Updater release resolver
 │
 │ заново читает repositories.<service>.url из Kernel Register
 │ и выбирает ровно запрошенный stable release
 ▼
Signed release assets
 │
 ├─ <service>-release.json
 ├─ <service>-release.json.sig.json
 └─ <service>-compose.tar.gz
 ▼
Updater verification
 │
 ├─ проверяет RSA-PSS-SHA256 подпись manifest
 │    trust anchor:
 │    /etc/exocortex/release-trust/<service>.pem
 │
 ├─ связывает service, tag и version
 ├─ проверяет minimum_updater_version
 ├─ проверяет checksum deployment bundle
 ├─ проверяет разрешённое содержимое Compose bundle
 ├─ требует immutable image digest sha256:...
 ├─ для Saturn также проверяет digest отдельного web image
 └─ запрещает downgrade и same-version replacement
 ▼
Подготовка замены
 │
 ├─ запоминает текущие image и version
 ├─ docker pull <image-reference>@sha256:<digest>
 ├─ для Saturn также docker pull <web-image>@sha256:<digest>
 ├─ сохраняет deployment snapshot:
 │    /var/lib/updater/deployments/<job-id>/deployment.json
 ├─ применяет только разрешённые deployment-файлы
 └─ атомарно обновляет image/version variables в env
 ▼
Docker Compose
 │
 ├─ обычный сервис:
 │    docker compose up -d --no-deps <compose-service>
 │
 └─ Saturn:
      docker compose stop api worker web
      docker compose run --rm --no-deps migrate
      docker compose up -d --no-deps api worker web
 ▼
Health verification
 │
 ├─ проверяет UPDATER_LOCAL_HEALTH_URL
 ├─ при наличии проверяет UPDATER_PUBLIC_HEALTH_URL
 ├─ сверяет reported runtime version с target version
 └─ сверяет реально запущенный immutable image
 ▼
COMPLETED
 │
 ├─ job сохраняет installed_version и installed_image
 ├─ UI обновляет фактическую версию сервиса
 └─ UI запускает новый Check for updates
```

Persistent volumes не удаляются. Updater заменяет только зарегистрированный
Compose-сервис и не выполняет произвольные команды, переданные приложением.

## 6. Автоматический и ручной rollback

### 6.1 Автоматический rollback в рамках текущего процесса

```text
Ошибка после MutationStarted
 │
 ▼
Updater
 │
 ├─ переводит job в ROLLING_BACK
 ├─ использует исходный ZIP, который ещё находится в RAM
 ├─ восстанавливает deployment snapshot
 ├─ возвращает прежние image/version values
 ├─ запускает предыдущий контейнер
 ├─ восстанавливает логические данные сервиса
 └─ проверяет прежнюю version и local health
 ▼
Результат
 │
 ├─ всё восстановлено → ROLLED_BACK
 └─ runtime или данные не восстановлены → ROLLBACK_FAILED
```

Варианты восстановления данных:

- стандартный сервис вызывает зарегистрированный внутренний
  `UPDATER_RESTORE_URL` с токеном конкретного head;
- Laboratory использует offline restore CLI из проверенного candidate image;
- Saturn останавливает `api`, `worker`, `web`, выполняет offline recovery CLI в
  одноразовом контейнере и только затем запускает предыдущие образы;
- временный путь для restore создаётся в tmpfs и удаляется после операции.

### 6.2 Rollback после рестарта Updater/хоста

Обычный ZIP не сохраняется Updater. После рестарта исходных байтов в RAM уже
нет, поэтому оператор должен выбрать сохранённый перед обновлением архив.

```text
Settings → завершённый/прерванный job → Rollback
 │
 │ пользователь выбирает исходный pre-update ZIP
 │ и подтверждает замену текущих данных
 ▼
Backend сервиса
 │
 └─ POST /v2/jobs/<job-id>/rollback
      {
        "filename":"<original-name>.zip",
        "sha256":"<sha256>",
        "data_base64":"<saved-zip>"
      }
 ▼
Updater
 │
 ├─ сверяет ZIP с backup_sha256 исходного job
 ├─ отклоняет любой другой архив
 ├─ временно размещает проверенные байты в tmpfs, если restore нужен path
 └─ выполняет тот же rollback pipeline
```

Обычная перезагрузка хоста сама по себе не требует restore: контейнеры и
persistent volumes остаются установленными. Копия нужна только для rollback или
восстановления данных прерванного update job.

## 7. Обновление общих helper-компонентов

Updater, Neptune, Gryphon и Wyvern являются общими компонентами хоста. Для них
нет backup gate приложения: они не заменяют данные конкретного сервиса.

```text
Settings → карточка нужного helper
 │
 ├─ POST <service-update-api>/check
 │    {"component":"updater|neptune|gryphon|wyvern"}
 │
 └─ Updater: POST /v2/check
      {
        "head_id":"<registered-head-id>",
        "component":"<helper>"
      }
 ▼
Окно Install <version>
 │
 │ предупреждает, что helper общий для хоста
 │ для Wyvern требуется confirm_shared = true
 │ backup ZIP приложения не создаётся
 ▼
Backend сервиса
 │
 └─ POST /v2/components/<helper>/updates
      {
        "head_id":"<registered-head-id>",
        "version":"<exact-version>",
        "request_id":"<uuid>",
        "confirm_shared":true   // Wyvern
      }
 ▼
Updater
 │
 ├─ проверяет, что head действительно потребляет helper
 ├─ повторно разрешает exact signed release
 ├─ создаёт durable component job
 └─ выполняет component-specific replacement
```

Дальнейшие ветки:

```text
Updater self-update
 │
 ├─ сохраняет предыдущий binary
 ├─ атомарно устанавливает новый binary
 ├─ запускает отдельный systemd supervisor
 ├─ перезапускает updater.service
 ├─ проверяет /v1/health через Unix socket
 └─ при ошибке возвращает предыдущий binary и снова запускает unit

Neptune
 │
 ├─ проверяет signed manifest и SHA-256 artifact
 ├─ атомарно заменяет neptuned и managed systemd unit
 ├─ daemon-reload + restart neptune.service
 ├─ проверяет /v1/health через /run/neptune/neptuned.sock
 └─ при ошибке восстанавливает binary/unit и перезапускает сервис

Gryphon
 │
 ├─ проверяет signed manifest и SHA-256 application bundle
 ├─ атомарно заменяет /usr/local/lib/gryphon/app и managed unit
 ├─ перезапускает gryphon.service
 ├─ проверяет health через /run/gryphon/client.sock
 └─ при ошибке возвращает предыдущие app/unit

Wyvern
 │
 ├─ проверяет подписанный release и фиксированный deployment profile
 ├─ заменяет только разрешённые container/systemd/config части gateway
 ├─ перезапускает wyvern.service
 ├─ проверяет readiness и установленную версию
 └─ при ошибке восстанавливает предыдущий проверенный deployment
```

## 8. Особый поток обновления Mastermind

Mastermind обновляется одной подписанной группой: `core`, `runtime`, `worker`.
Обычная передача ZIP в base64 не применяется, потому что зашифрованный архив
может быть существенно больше 128 МиБ.

```text
Mastermind Settings
 │
 │ пользователь выбирает exact group release
 ▼
Mastermind Core
 │
 └─ POST /v1/heads/<head-id>/preparations
      {
        "request_id":"<request-id>",
        "version":"<target-version>"
      }
 ▼
Updater preparation
 │
 ├─ разрешает и проверяет подписанный Mastermind manifest
 ├─ проверяет saved_copy_protocol = 2
 ├─ проверяет группу Core + Runtime + Worker
 ├─ проверяет schema/Bridge/Obsidian/model compatibility
 ├─ проверяет Compose profile
 └─ заранее docker pull всех трёх immutable image digests
 ▼
Mastermind writer barrier
 │
 ├─ блокирует новые записи
 ├─ создаёт согласованный snapshot
 ├─ формирует стандартный зашифрованный ZIP
 ├─ вычисляет size и SHA-256
 └─ предоставляет одноразовое скачивание оператору
 ▼
Браузер
 │
 ├─ сохраняет ZIP на компьютер
 ├─ снова выбирает именно этот файл
 ├─ хеширует его потоком
 └─ отправляет без полного buffering в память
 ▼
Updater volatile spool
 │
 ├─ POST /v1/heads/<head-id>/backup-spools
 │    {request_id, filename, size, sha256}
 │
 ├─ PUT  /v1/heads/<head-id>/backup-spools/<spool-id>/content
 │    Content-Type: application/zip
 │    Content-Length: <exact-size>
 │
 └─ POST /v1/heads/<head-id>/backup-spools/<spool-id>/seal
      └─ Updater повторно проверяет size и SHA-256
 ▼
Sealed spool
 │
 │ находится только в private tmpfs под /dev/shm
 │ disk fallback отсутствует
 │ лимит протокола — до 8 ГиБ, дополнительно ограничен
 │ свободным местом tmpfs и обязательным reserve
 ▼
Mastermind Core
 │
 │ создаёт exocortex.update-backup.v2 receipt,
 │ связывающий request/head/version/spool hash/size
 │
 └─ POST /v2/updates
      {
        "request_id":"<request-id>",
        "head_id":"<head-id>",
        "service":"mastermind",
        "version":"<target-version>",
        "preparation_id":"<preparation-job-id>",
        "operator_saved":true,
        "backup_receipt":"<signed-receipt>",
        "backup":{"spool_id":"<sealed-spool-id>"}
      }
 ▼
Updater Mastermind apply
 │
 ├─ убеждается, что подготовленный release не изменился
 ├─ вызывает Core:
 │    POST /api/internal/updater/confirm
 │    └─ Core подтверждает retained writer barrier и source schema
 ├─ сохраняет предыдущие 3 image digests и deployment snapshot
 ├─ останавливает core, runtime, worker
 ├─ применяет новый Compose и signed release metadata
 ├─ обновляет MASTERMIND_*_IMAGE и MASTERMIND_VERSION
 ├─ запускает migration одноразовым Core container
 ├─ сначала запускает и ждёт runtime + worker
 └─ затем запускает core
 ▼
Mastermind functional acceptance
 │
 ├─ local health
 ├─ POST /api/internal/updater/functional
 ├─ version и database schema
 ├─ Runtime/Bridge version
 ├─ Worker availability
 ├─ Vault availability
 ├─ Obsidian version
 ├─ model SHA-256
 └─ optional public health
 ▼
COMPLETED
 │
 ├─ Core освобождает writer barrier только после наблюдения COMPLETED
 └─ terminal job удаляет volatile spool
```

Если после mutation возникает ошибка, Updater останавливает всю группу,
возвращает предыдущие Compose/env/manifest и три image digest, восстанавливает
данные из sealed ZIP до запуска старого Core и повторяет functional acceptance.
Результат — `ROLLED_BACK` или `ROLLBACK_FAILED`.

После рестарта transient spool исчезает. Для recovery оператор снова выбирает
исходный ZIP, Mastermind загружает и sealing-ит новый spool и вызывает:

```text
POST /v2/jobs/<job-id>/rollback-saved-spool
{
  "spool_id":"<new-sealed-spool-id>",
  "operator_saved":true
}
```

Rollback уже успешно установленной версии Mastermind — новая контролируемая
операция: создаётся свежий backup текущих данных и новый writer barrier. Это не
повторное использование старого transient spool.

## 9. API маршруты по проектам

| Проект | Application-side update API | Особенности |
| --- | --- | --- |
| Kernel | `/api/update-flow/*` | Общий overlay; service + Updater + Neptune |
| Volt | `/api/v1/update-flow/*` | Общий overlay; service + Updater + Neptune |
| Chronos | `/api/update-flow/*` | Python-реализация того же saved-copy protocol; также Gryphon |
| Laboratory | `/api/update-flow/*` | Общий overlay; Updater + Neptune + Wyvern; offline restore |
| Saturn | `/operator/updates/flow/*` | Собственный NestJS controller; отдельные api/worker/web images и migration |
| Perimetr | `/v1/updater/*` | Собственные `check`, `prepare`, `install`, `jobs`, `rollback`; тот же `/v2` Updater protocol |
| Mastermind | `/api/owner/updates/*` и `/api/owner/helper-updates/*` | Group preparation, writer barrier, encrypted saved copy и streaming spool |

Типовой набор application-side маршрутов выглядит так:

```text
POST <base>/check
POST <base>/backup
POST <base>/install/<component>
GET  <base>/jobs
GET  <base>/jobs/<job-id>
POST <base>/jobs/<job-id>/rollback
```

Маршруты приложений не являются привилегированным установщиком. Они только
авторизуют оператора, строят backup/receipt и проксируют строго типизированный
запрос в локальный Updater.

## 10. Хранение и восстановление состояния

```text
/etc/exocortex/updater-heads.json
  └─ root-owned список зарегистрированных heads и путей к env

/etc/exocortex/release-trust/<component>.pem
  └─ локально закреплённые public keys для release manifests

/var/lib/updater/jobs/<job-id>.json
  └─ durable state, progress, target, previous version/image, backup hash

/var/lib/updater/deployments/<job-id>/deployment.json
  └─ snapshot разрешённых deployment metadata для rollback

/var/lib/updater/register-<head-id>.json
  └─ проверенный cache Kernel Register

RAM
  └─ обычный pre-update ZIP до завершения update/automatic rollback

/dev/shm/...
  └─ только transient sealed spool Mastermind и временный restore input
```

По умолчанию Updater удерживает не более 20 terminal jobs и удаляет terminal
metadata старше 30 дней. ZIP приложения, `.env` с секретами и произвольные
копии данных в job storage не сохраняются.

## 11. Реализация в исходниках

Основные точки входа:

- `updater/docs/UPDATE-PROTOCOL.md` — saved-copy protocol 2;
- `updater/docs/saved-group-updates.md` — групповой протокол Mastermind;
- `updater/internal/api/protocol.go` — `/v2/check`, `/v2/updates`, rollback;
- `updater/internal/api/component_updates.go` — helper updates;
- `updater/internal/api/spool.go` — Mastermind backup spool;
- `updater/internal/engine/engine.go` — обычная установка, health и rollback;
- `updater/internal/engine/mastermind_group.go` — применение группы Mastermind;
- `updater/internal/release/release.go` — выбор и проверка релиза;
- `updater/internal/releaseauth/verify.go` — проверка RSA-PSS подписи;
- `kernel/src/update-overlay.js` — общий browser workflow;
- `kernel/server/update-flow.js` и `kernel/server/update-backup.js` — общий backend contract;
- `volt/server/update-flow.js` — интеграция Volt;
- `chronos/app/update_flow.py` — интеграция Chronos;
- `laboratory/services/api/src/update-flow.js` — интеграция Laboratory;
- `saturn/apps/api/src/updater.controller.ts` — интеграция Saturn;
- `perimetr/app/operations_api.py` — интеграция Perimetr;
- `mastermind/src/mastermind/updates.py` — writer barrier, saved copy и spool.
