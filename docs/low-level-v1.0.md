# v1.0 — голос, placement, биллинг

Спецификация сборки недель 1–5. Решения и границы системы — в [RFC-001](architecture-rfc.md). Этот документ говорит, что именно собирается в первой версии.

## 1. Результат

Пользователь открывает сайт, входит по письму, проходит placement, покупает или тратит пробные минуты, говорит с ментором по сценарию и после звонка видит отчёт: ошибки и новые слова. Слова лежат в словаре, повторять их ещё негде. Это появится в v1.1.

Готово, когда на проде одновременно держатся десятки звонков, p50 задержки до первого звука ниже 1.2 с на сценарии из коротких реплик, повторный `stop` не списывает секунды дважды, а обрыв сокета не теряет уже сказанные реплики.

## 2. Процессы

Микросервисов нет. На машине четыре контейнера приложения и данных плюс Caddy.

| Процесс | Образ | Слушает | Пишет |
| :--- | :--- | :--- | :--- |
| `caddy` | Caddy 2 | 443 наружу | статику `web/dist`, прокси на `core` |
| `core` | Go | 8080 только в docker-сети | Postgres, Redis, Deepgram, OpenAI, Cartesia, ЮKassa, Resend |
| `ai-worker` | Python, ARQ | ничего снаружи | Postgres, Redis, OpenAI |
| `postgres` | 16 | только docker-сеть | том |
| `redis` | 7, AOF, `noeviction` | только docker-сеть | том |

Репозиторий один: `web/`, `core/`, `ai/`, `deploy/`. Миграции — goose в `core`. Воркер схему не меняет.

Внутри `core` пакеты, не сервисы: `auth`, `billing`, `voice`, `placement`, `catalog` (сценарии), `outbox`. Общий Postgres.

## 3. Клиент

Vite, React, TypeScript. Роутер: `/login`, `/onboarding`, `/placement`, `/practice`, `/call/:id`, `/call/:id/report`, `/vocab`, `/billing`.

Микрофон:

* `getUserMedia({ audio: { echoCancellation: true, noiseSuppression: true, autoGainControl: true, channelCount: 1 } })`.
* `AudioWorklet` режет вход в PCM s16le, 16 кГц, кадр 20 мс (640 байт).
* Пока в очереди плеера есть звук ментора, кадры в сокет не уходят.
* Порог энергии на входе во время речи ментора: клиент останавливает плеер и шлёт `barge_in`, после этого кадры снова идут.
* Плеер — `AudioContext`, PCM 24 кГц, стартовый буфер 60 мс.

Часовой пояс берётся из `Intl.DateTimeFormat().resolvedOptions().timeZone` и один раз сохраняется в профиль.

## 4. Вход и профиль

Письмо с кодом через Resend. Пароля нет. Сессия браузера — случайный токен в httpOnly-куке `Secure`, `SameSite=Lax`, запись в `browser_sessions` (хранится хеш). Срок 30 дней, продлевается при заходе.

```
POST /v1/auth/magic-link
{ "email": "a@b.c" }
→ 202
```

Ответ всегда 202, чтобы по коду не перебирать зарегистрированные адреса. Не больше 5 писем в час на адрес.

```
POST /v1/auth/verify
{ "email": "a@b.c", "code": "123456" }
→ 200 + Set-Cookie
{ "user": { "id": "...", "needs_onboarding": true } }
```

```
POST /v1/auth/logout
GET  /v1/me
PATCH /v1/me
{ "timezone": "Europe/Moscow", "native_language": "ru",
  "learning_goal": "it", "target_level": "B2" }
DELETE /v1/me
```

`DELETE /v1/me` удаляет пользователя и все его звонки, реплики, отчёты, слова, проводки журнала. Заказы оплаты остаются обезличенными: `user_id` обнуляется, сумма и id провайдера сохраняются.

Новый аккаунт получает одну проводку `grant` на 300 секунд, ключ `signup:{user_id}`.

## 5. Placement

Десять закрытых вопросов из `placement_items` (сид в миграции, по два на A1–B2 и по одному на C1–C2, навык grammar или vocab). Модель на эту часть не вызывается.

```
POST /v1/placement
→ { "assessment_id": "...", "items": [ { "id", "prompt", "options": ["..."] } ] }

POST /v1/placement/{id}/answers
{ "answers": [ { "item_id": "...", "choice": 1 } ] }
→ { "speaking_required": true }
```

Дальше обязательный звонок 60–90 секунд со сценарием `slug=placement`. Потолок сессии для этого сценария 90 секунд, не 15 минут. Воркер по транскрипту и ключам закрытых вопросов пишет `level_assessments.assigned_level` и выставляет `users.current_level`, если уровень ещё пуст. Пока `analysis_status != done`, клиент опрашивает:

```
GET /v1/placement/{id}
→ { "status": "pending|done|failed", "assigned_level": "A2" }
```

Опрос раз в 2 секунды, не дольше 60 секунд. При `failed` уровень не ставится, пользователь может перезапустить placement один раз.

`score_breakdown`:

```json
{
  "closed_correct": 7,
  "closed_total": 10,
  "speaking_cefr": "A2",
  "speaking_note": "Короткие фразы, путает he/she."
}
```

`item_results` хранит выбранные индексы. Это черновая маршрутизация, в интерфейсе так и написано.

## 6. Каталог и звонок

```
GET /v1/scenarios
→ [{ "id", "slug", "title", "topic", "min_level", "max_level" }]
```

Скрытые и те, чей `min_level` выше текущего, не отдаются. До placement список пуст.

```
POST /v1/calls
{ "scenario_id": "..." }
→ 201 { "call_id": "...", "ticket": "...", "expires_in": 30 }
```

Отказ `402`, если доступный баланс меньше 30 секунд. Отказ `409`, если у пользователя уже есть `call_sessions.status=active`.

Тикет одноразовый, живёт 30 секунд в Redis.

### WebSocket

`GET wss://{host}/v1/calls/stream?ticket={ticket}`

Первый кадр от сервера — JSON `ready`. Дальше клиент шлёт JSON `start` и бинарные кадры. Текст — JSON text frame, звук — binary frame.

Клиент → сервер:

```json
{ "type": "start" }
{ "type": "barge_in" }
{ "type": "stop" }
```

Бинарный кадр: ровно 640 байт PCM s16le 16 кГц. Иные размеры сервер закрывает кодом `bad_audio`.

Сервер → клиент:

```json
{ "type": "ready", "call_id": "...", "sample_rate": 24000 }
{ "type": "transcript", "final": false, "text": "I goes" }
{ "type": "transcript", "final": true, "text": "I go to work", "seq": 1 }
{ "type": "assistant_text", "text": "Got it.", "seq": 2, "partial": true }
{ "type": "audio_end", "seq": 2 }
{ "type": "stop_playback" }
{ "type": "balance", "seconds_left": 240 }
{ "type": "call_ended", "billed_seconds": 86, "end_reason": "client_stop" }
{ "type": "error", "code": "provider_down" }
```

Бинарный кадр сервера: PCM s16le 24 кГц, произвольная длина чанка Cartesia.

Коды закрытия: `4001` тикет, `4002` нет баланса, `4003` плохой звук, `4004` потолок 15 минут, `4010` провайдер.

### Шаг одного хода

1. Кадры пользователя уходят в уже открытый сокет Flux: `wss://api.deepgram.com/v2/listen?model=flux-general-en&encoding=linear16&sample_rate=16000`. Пороги: `eot_threshold=0.7`, `eager_eot_threshold=0.5`, `eot_timeout_ms=3000`. Язык английский, отдельный `language` не передаётся.
2. `EagerEndOfTurn` с непустым текстом запускает LLM, если генерация ещё не идёт.
3. `TurnResumed` отменяет LLM и Cartesia, если финальный текст ещё не записан.
4. `EndOfTurn` пишет `messages` (`role=user`, следующий `seq`) и обновляет хвост в Redis.
5. Запрос к OpenAI, Chat Completions, `model=gpt-6-luna`, `stream=true`, `reasoning_effort=none`, `max_tokens=180`. Системный промпт: обёртка версии `tutor_v1` + `scenarios.system_prompt` + уровень + цель + последние 500 символов прошлого `feedback_reports.summary` + хвост из 16 реплик этого звонка.
6. Первый кусок на синтез — 12 токенов или текст до первой запятой, точки, `?`, `!`, что наступит раньше. Дальше чанки по границе слова, `continue=true`, тот же `context_id`.
7. Cartesia: `wss://api.cartesia.ai/tts/websocket?cartesia_version=2026-08-14`, `model_id=sonic-3.6-2026-08-27`, `language=en`, контейнер `raw`, `pcm_s16le`, 24000. Голос один, id в конфиге `CARTESIA_VOICE_ID`.
8. Финальный текст ментора пишется в `messages` целиком, когда поток LLM закончился. Частичные куски в базу не кладутся.
9. `StartOfTurn` во время ответа: `cancel` контекста Cartesia, клиенту `stop_playback`, недописанный ответ ментора сохраняется как есть.

Хвост Redis: ключ `call:{id}:tail`, TTL 2 часа. Это копия, источник правды — `messages`.

## 7. Биллинг

Баланс считается только из `usage_ledger`. `users.balance_seconds` обновляется в той же транзакции, что и проводка.

Причины: `grant`, `reserve`, `tick`, `settle`, `refund`.

На `POST /v1/calls`:

* `available = balance_seconds`.
* Резерв `min(available, 900)` секунд, ключ `reserve:{call_id}`.
* Проекция уменьшается на резерв сразу, чтобы второй звонок не увидел те же секунды.

Каждые 10 секунд живого сокета: `tick` на −10, ключ `tick:{call_id}:{n}`. Когда сумма тиков догоняет резерв, сервер шлёт `call_ended` с `end_reason=balance`.

На завершении, включая обрыв сокета:

* `billed = min(резерв, фактические секунды сокета)`.
* Если тиков списано меньше `billed`, дописывается один `tick` на разницу.
* Неиспользованный резерв возвращается проводкой `settle` с положительной дельтой, ключ `settle:{call_id}`.
* Повторный settle находит ключ и ничего не пишет.

Длительность считает сервер по часам между `ready` и закрытием сокета. Поле от клиента игнорируется, его в протоколе нет.

Пакеты ЮKassa, суммы в конфиге, на старте два: 30 минут и 120 минут.

```
POST /v1/billing/checkout
{ "pack_code": "m30" }
→ { "confirmation_url": "..." }

POST /v1/billing/yookassa/webhook
```

Webhook проверяет подпись и идемпотентен по `payment_orders.provider_payment_id`. Успех пишет `grant` с ключом `pay:{provider_payment_id}`.

```
GET /v1/billing
→ { "balance_seconds": 840, "ledger": [ { "delta_seconds", "reason", "created_at" } ] }
```

## 8. После звонка

В транзакции завершения звонка: статус `completed` или `abandoned`, `outbox` со статусом `pending`, ключ `process_call:{call_id}`.

Релей раз в секунду забирает pending, кладёт задачу ARQ `process_call`, ставит `sent`. Падение между commit и Redis оставляет строку pending, следующий проход её догоняет.

Воркер:

1. Если `feedback_reports` на этот `call_id` уже есть, выходит.
2. Ставит `analysis_status=running`.
3. Зовёт `gpt-6.1-terra`, `reasoning_effort=low`, structured output.
4. В одной транзакции пишет отчёт, делает upsert слов и ставит `analysis_status=done`. Сбой откатывает транзакцию, статус `failed`, ARQ повторяет до 3 раз.

Слова: конфликт `(user_id, lemma, pos)` не сбрасывает ease и интервал. Если перевод пуст, он дописывается. Новая строка создаётся с `ease_factor=2.5`, `repetitions=0`, `interval_days=0`, `next_review_at` на начало завтрашнего дня в часовом поясе пользователя. В день звонка слово не попадает в очередь повторений.

Задания `daily_tasks` этот воркер не создаёт.

Контракт судьи:

```json
{
  "grammar_score": 0.0,
  "summary": "Коротко по-русски, до 500 символов",
  "mistakes": [
    {
      "kind": "grammar",
      "span": "he go",
      "correction": "he goes",
      "explanation_ru": "В третьем лице нужно -s.",
      "cefr": "A1"
    }
  ],
  "vocabulary": [
    {
      "lemma": "commute",
      "pos": "verb",
      "translation": "ездить на работу",
      "context_sentence": "I commute by train."
    }
  ]
}
```

`grammar_score` — число от 0 до 1. В отчёт берётся не больше 8 ошибок и 8 слов. `pos` один из `noun|verb|adj|adv|phrase`.

```
GET /v1/calls/{id}
→ {
  "status": "completed",
  "analysis_status": "done",
  "billed_seconds": 86,
  "messages": [ { "seq", "role", "content" } ],
  "report": { "grammar_score", "summary", "mistakes", "vocabulary" }
}

GET /v1/vocab
→ [ { "lemma", "pos", "translation", "context_sentence", "next_review_at" } ]
```

Словарь в v1.0 только для чтения.

Себестоимость пишется в `call_sessions.cost_usd` из фактических usage провайдеров, если они пришли в ответе. Нет usage — оценка: секунды аудио в Flux × список $0.0077/60, символы в Cartesia по кредиту тарифа, токены Luna по $0.10 / $0.50 за миллион. Terra добавляется воркером отдельным слагаемым в ту же колонку.

## 9. Таблицы, которые создаёт v1.0

Остальные таблицы канона из RFC в этой миграции не создаются, кроме колонок словаря под SM-2: они есть сразу, чтобы v1.1 не переписывал таблицу. Поля ease и интервал в v1.0 никто не меняет после вставки.

`users`, `login_codes`, `browser_sessions`, `usage_ledger`, `payment_orders`, `scenarios`, `call_sessions`, `messages`, `feedback_reports`, `vocabulary`, `placement_items`, `level_assessments`, `outbox`.

Индексы, без которых ломается целостность, а не скорость: unique на email, тикет не в Postgres, unique `usage_ledger.idempotency_key`, unique `payment_orders.provider_payment_id`, unique `(call_session_id, seq)`, unique `feedback_reports.call_session_id`, unique `(user_id, lemma, pos)`, unique `outbox.idempotency_key`.

## 10. Инфраструктура

Цифры канала, диска и месячного сценария на 1000 DAU собраны в [расчёте ёмкости](capacity-and-cost.md). Ниже только то, что нужно поднять.

Аренда обязательна для приёмки, не для первого коммита. До прода разработка идёт на локальном Compose.

Прод, неделя 1:

* VPS 4 vCPU, 8 GB RAM, Хельсинки. Ориентир — Hetzner CPX31.
* Домен, A-запись на этот адрес.
* Бакет объектного хранилища. Ночной `pg_dump` с ротацией 14 копий. Аудио звонков туда не кладётся.
* Секреты в файле на сервере, в git только `deploy/.env.example`.

Наружу открыт 443. Деплой — GitHub Actions по SSH: сборка образов на VPS, `docker compose up -d`. Отдельный registry не арендуется.

Redis: `appendonly yes`, `maxmemory 512mb`, `maxmemory-policy noeviction`. Очередь и сессии не должны вытесняться.

Наблюдаемость без отдельного вендора. Логи обоих процессов в stdout, поле `call_id` на каждом голосовом шаге. Гистограмма `voice_first_audio_ms` (EagerEndOfTurn → первый байт в сокет) и `voice_barge_in_ms` пишется в лог и в `call_sessions` не требуется: достаточно логов, чтобы в конце недели посчитать p50/p95. PostHog, проект в регионе EU:

* `signed_up`, `placement_done`, `call_started`, `call_ended` (секунды, `end_reason`), `report_viewed`, `checkout_started`, `paid`.

Сервер дублирует `call_ended` и `paid`, чтобы удержание не зависело от закрытой вкладки.

## 11. Вне этого релиза

План, SM-2-показы, ежедневные задания, стрик, XP, аудирование, экзамен, хранение сырого аудио, Silero, второй сервер, Kubernetes.
