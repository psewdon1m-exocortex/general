Сервис
  │ локально знает:
  │   KERNEL_URL
  │   KERNEL_SERVICE_TOKEN
  │
  │ Authorization: Bearer <KERNEL_SERVICE_TOKEN>
  │ POST /api/v1/register/resolve
  │ {
  │   "keys": [
  │     "services.<service>.<variable>"
  │   ]
  │ }
  ▼
Kernel
  │ открывает текущую опубликованную ревизию Register
  │
  │ находит соответствие:
  │
  │ services.<service>.<variable>
  │        ↓
  │ volt://<entry-id>/<field-id>
  │
  │ проверяет строгий формат ссылки
  │ группирует и удаляет повторяющиеся ссылки
  │ разбивает запросы к Volt на пакеты до 20 ссылок
  │
  │ использует собственные:
  │   VOLT_URL
  │   VOLT_KERNEL_TOKEN
  │
  │ Authorization: Bearer <VOLT_KERNEL_TOKEN>
  │ POST /api/v1/internal/kernel/resolve
  │ {
  │   "references": [
  │     "volt://<entry-id>/<field-id>"
  │   ]
  │ }
  ▼
Volt
  │ доверяет только Kernel
  │
  │ для каждой ссылки:
  │   1. проверяет entry-id
  │   2. проверяет field-id
  │   3. находит актуальную ревизию записи
  │   4. расшифровывает поле
  │   5. определяет visibility: plain или secret
  │
  │ возвращает Kernel:
  │ {
  │   "volt://<entry-id>/<field-id>": {
  │     "value": "<actual-value>",
  │     "revision": 3,
  │     "visibility": "plain | secret"
  │   }
  │ }
  ▼
Kernel
  │ связывает полученные значения с исходными Register keys
  │
  │ НЕ сохраняет фактические значения:
  │   - в Register
  │   - в Register revisions
  │   - в cache
  │   - в audit
  │   - в backup
  │
  │ возвращает:
  │ {
  │   "schema": "exocortex.register.resolution.v1",
  │   "register_revision": "register-...",
  │   "values": {
  │     "services.<service>.<variable>": {
  │       "value": "<actual-value>",
  │       "secret": false,
  │       "volt_revision": 3
  │     }
  │   }
  │ }
  ▼
Сервис
  │ получает только запрошенные значения
  │ хранит их только в памяти процесса
  ▼
Использует значение