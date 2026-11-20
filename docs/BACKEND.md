# ADOLAT — Backend: спецификация

Владелец: Участник 3 (API, модели, игровой движок, аналитика), Участник 2 (`legal/`, `rag/`, `ai/`).
Связанные документы: [ARCHITECTURE.md](../ARCHITECTURE.md), [FRONTEND.md](FRONTEND.md), [INGESTION.md](INGESTION.md).

---

## 1. Стек и запуск

- Python 3.12, FastAPI, Pydantic v2, SQLAlchemy 2 (sync, psycopg 3), Alembic
- PostgreSQL 16 + pgvector, pg_trgm
- `anthropic` SDK, `sse-starlette`
- pytest

```
docker compose up backend           # :8000, Swagger: /docs, схема: /openapi.json
make test                           # pytest
make api-client                     # регенерировать TS-клиент для фронта
```

Переменные окружения (`.env.example`):

| Переменная | Назначение |
|---|---|
| `DATABASE_URL` | Postgres |
| `ANTHROPIC_API_KEY` | необязательна; пустая = режим без AI |
| `LLM_MODEL_MENTOR` | `claude-sonnet-5-5` |
| `LLM_MODEL_FAST` | `claude-haiku-5-5` (NPC, классификация запросов) |
| `EMBEDDER` | выбранная модель эмбеддингов (см. INGESTION.md) |
| `CORS_ORIGINS` | адрес фронтенда |
| `RAG_MIN_SCORE` | порог релевантности для отказа «недостаточно источников» |

---

## 2. Почему REST + SSE, а не WebSocket

- Игра пошаговая: игрок делает действие, сервер отвечает новым состоянием. Это обычный запрос и ответ, постоянное соединение тут ничего не даёт.
- Сервер ничего не присылает сам по себе. Нет мультиплеера и нет таймеров на сервере.
- Единственное место, где нужен поток, — ответ AI-наставника. Для этого хватает **SSE**: односторонний поток, работает поверх обычного HTTP, проще проксировать и отлаживать.
- WebSocket можно добавить позже (например, для режима «преподаватель смотрит группу онлайн»). В MVP его нет.

---

## 3. Общие правила API

- Префикс `/api`. Формат JSON, кодировка UTF-8.
- Язык: query-параметр `locale` ∈ `ru | uz-Latn | uz-Cyrl`. Если не передан, берётся язык сессии, а если сессии нет — `ru`.
- Все тексты для пользователя сервер отдаёт уже переведёнными. Фронт не хранит тексты сценариев.
- Формат ошибки (всегда один и тот же):

```json
{ "error": { "code": "ILLEGAL_ACTION", "message": "Действие недоступно на этом этапе", "details": {} } }
```

| HTTP | code | Когда |
|---|---|---|
| 404 | `NOT_FOUND` | нет сценария, сессии, citation |
| 409 | `ILLEGAL_ACTION` | действие недоступно в текущем состоянии |
| 409 | `SESSION_FINISHED` | действие после завершения дела |
| 409 | `SESSION_NOT_FINISHED` | запрос результата до конца игры |
| 422 | `VALIDATION_ERROR` | неверное тело запроса |
| 503 | `DB_UNAVAILABLE` | нет соединения с базой |

Недоступность LLM — **не ошибка**. Эндпоинт отвечает fallback-содержимым и флагом `ai_unavailable: true`.

---

## 4. Общие объекты (Pydantic → OpenAPI)

```jsonc
// CitationView: всё, что фронт показывает о статье закона
{
  "citation_id": "CPRL.art.X",
  "document_title": "Закон Республики Узбекистан «О защите прав потребителей»",
  "document_type": "law",
  "document_number": "...",
  "article_number": "X",
  "article_title": "...",
  "part": null,
  "text": "дословный текст статьи из выбранной редакции",
  "language": "ru",
  "is_official_text": true,
  "edition": {
    "effective_from": "2024-01-01",
    "effective_to": null,
    "status": "current"              // current | historical | unconfirmed
  },
  "source_url": "https://lex.uz/docs/...",
  "last_verified_at": "2026-11-21",
  "translation_note": null           // текст, если показан не официальный язык
}

// ActionView: кнопка, которую фронт может показать
{
  "action_id": "present_receipt",
  "kind": "evidence",                // question | evidence | motion | objection | argument | decision | continue
  "label": "Представить кассовый чек",
  "target_character": "judge",       // для question: кому вопрос
  "evidence_id": "receipt",          // для evidence
  "options": null,                   // для decision: [{ "id": "refund", "label": "..." }]
  "hint_citation_ids": ["CPC.art.Z"] // можно открыть справку до выбора
}

// GameEvent: что произошло после действия, фронт проигрывает это по очереди
{ "type": "say",            "speaker": "judge", "text": "...", "emotion": "neutral" }
{ "type": "react",          "character": "defendant", "reaction": "object" }      // nod | object | surprised | think
{ "type": "stage_changed",  "stage": "DEBATE", "title": "Судебные прения" }
{ "type": "evidence_added", "evidence_id": "expert_report" }
{ "type": "score",          "delta": 10, "total": 40 }
{ "type": "legal_note",     "citation_id": "CPC.art.Z", "text": "Почему это важно..." }
{ "type": "mistake",        "category": "evidence", "text": "..." }   // показывается только в Results, если scenario.reveal_mistakes = false

// GameState
{
  "session_id": "uuid",
  "scenario_id": "consumer_dispute_001",
  "locale": "ru",
  "stage": "EVIDENCE_EXAMINATION",
  "stage_title": "Исследование доказательств",
  "stage_index": 5, "stage_count": 9,
  "score": 40, "max_score": 100,
  "evidence": [{ "id": "receipt", "title": "Кассовый чек", "type": "document", "reviewed": true, "presented": false }],
  "available_actions": [ActionView],
  "events": [GameEvent],             // новые события после последнего действия
  "finished": false,
  "legal_date": "2026-03-01"
}
```

---

## 5. Эндпоинты

### 5.1 Служебные

#### `GET /api/health`
Что делает: проверяет базу (`SELECT 1`), число статей и chunks, загружены ли сценарии и есть ли ключ LLM (без реального вызова).
```jsonc
// 200 OK
{ "status": "ok", "db": true, "articles": 1834, "chunks": 4120, "scenarios": 1, "llm_configured": true, "embedder": "bge-m3" }

// 503 Service Unavailable (база недоступна)
{ "status": "degraded", "db": false, "articles": null, "chunks": null, "scenarios": 1, "llm_configured": true, "embedder": "bge-m3" }
```
Используется в чек-листе перед демо.

### 5.2 Пользователи

#### `POST /api/users`
Тело: `{ "display_name": "Азиз" }` (необязательно).
1. Создать запись в `users` с новым UUID.
2. Вернуть `user_id`. Фронт сохраняет его в localStorage.

Авторизации в MVP нет.

```jsonc
// 201 Created
{ "user_id": "6f1c2a9e-3b7d-4e21-9a55-0c8f4d2e7b10", "display_name": "Азиз", "created_at": "2026-11-21T09:14:03Z" }
```

#### `GET /api/users/{user_id}/dashboard?locale=`
1. Выбрать завершённые сессии пользователя и их итоги.
2. Выбрать попытки тестов, посчитать прирост знаний (post − pre) по каждому сценарию.
3. Посчитать уровень правовой грамотности: среднее нормированных баллов за игры и пост-тесты, 0–100.
4. Рекомендации: сценарии, которые пользователь ещё не прошёл, плюс темы из его самых частых категорий ошибок.
```jsonc
// 200 OK
{
  "user": { "user_id": "6f1c2a9e-3b7d-4e21-9a55-0c8f4d2e7b10", "display_name": "Азиз" },
  "knowledge_level": { "score": 62, "label": "Уверенный" },
  "completed": [
    { "session_id": "a3e1b0c4-5d2f-4c8e-8f71-2b9d6e0a1c33", "scenario_id": "consumer_dispute_001",
      "title": "Неисправный смартфон", "outcome": "claim_partially_granted", "outcome_title": "Иск удовлетворён частично",
      "score": 70, "max_score": 100, "finished_at": "2026-11-21T09:31:47Z" }
  ],
  "learning_gain": [
    { "scenario_id": "consumer_dispute_001", "pre": 4, "post": 8, "max": 10,
      "note": "Предварительный показатель, не доказанный эффект обучения" }
  ],
  "mistake_categories": [
    { "category": "evidence", "title": "Доказательства", "count": 2 },
    { "category": "remedy_choice", "title": "Выбор требования", "count": 1 }
  ],
  "recommended": [
    { "scenario_id": "consumer_dispute_002", "title": "Возврат товара надлежащего качества", "category": "Защита прав потребителей",
      "difficulty": "beginner", "minutes": 15, "topics": ["Обмен товара", "Сроки"], "available": false, "reason": "Скоро" }
  ]
}
```

### 5.3 Сценарии

#### `GET /api/scenarios?locale=`
Берёт сценарии из кэша в памяти и переводит карточки.
```jsonc
// 200 OK
[
  { "scenario_id": "consumer_dispute_001", "title": "Неисправный смартфон", "category": "Защита прав потребителей",
    "difficulty": "beginner", "minutes": 15, "topics": ["Качество товара", "Доказательства", "Требования покупателя"],
    "cover": "/game/covers/consumer_dispute_001.png",
    "fictional_notice": "Учебная ситуация, не реальное дело", "available": true }
]
```

#### `GET /api/scenarios/{scenario_id}?locale=`
Описание дела для экрана введения: история конфликта, участники (id, имя, роль, спрайт), требования сторон, список документов (только названия), `legal_date`, список применимых актов (названия + ссылки на lex.uz).
1. Взять сценарий из кэша → иначе `404 NOT_FOUND`.
2. Перевести тексты на `locale`.
3. Для применимых актов взять названия и `source_url` из `legal_documents`.

```jsonc
// 200 OK
{
  "scenario_id": "consumer_dispute_001",
  "title": "Неисправный смартфон",
  "fictional_notice": "Учебная ситуация, не реальное дело. Имена и обстоятельства вымышлены.",
  "legal_date": "2026-03-01",
  "story": "Покупатель приобрёл смартфон в магазине электроники. Через 10 дней устройство перестало заряжаться. Продавец отказался принять товар, сославшись на то, что поломка возникла по вине покупателя.",
  "player_role": { "character_id": "plaintiff_rep", "title": "Вы — представитель истца" },
  "characters": [
    { "id": "judge",          "name": "Судья",                   "role": "Судья",                         "sprite": "judge" },
    { "id": "plaintiff_rep",  "name": "Вы",                      "role": "Представитель истца",           "sprite": "lawyer_a" },
    { "id": "plaintiff",      "name": "Дилшод Каримов",          "role": "Истец, покупатель",             "sprite": "citizen_m" },
    { "id": "defendant_rep",  "name": "Нодира Юсупова",          "role": "Представитель ответчика",       "sprite": "lawyer_b" },
    { "id": "expert",         "name": "Эксперт",                 "role": "Специалист сервисного центра",  "sprite": "expert" }
  ],
  "claims": [
    { "side": "plaintiff", "text": "Расторгнуть договор и вернуть уплаченную сумму" },
    { "side": "defendant", "text": "Отказать в иске: недостаток возник по вине покупателя" }
  ],
  "documents": [
    { "id": "contract", "title": "Договор купли-продажи" },
    { "id": "receipt",  "title": "Кассовый чек" },
    { "id": "warranty", "title": "Гарантийный талон" },
    { "id": "chat",     "title": "Переписка с продавцом" },
    { "id": "claim_letter", "title": "Письменная претензия покупателя" }
  ],
  "legal_acts": [
    { "document_key": "CPRL", "title": "Закон Республики Узбекистан «О защите прав потребителей»", "source_url": "https://lex.uz/docs/<id>" },
    { "document_key": "CC",   "title": "Гражданский кодекс Республики Узбекистан",               "source_url": "https://lex.uz/docs/<id>" },
    { "document_key": "CPC",  "title": "Гражданский процессуальный кодекс Республики Узбекистан", "source_url": "https://lex.uz/docs/<id>" }
  ]
}
```

### 5.4 Тесты (до и после игры)

#### `GET /api/quiz/{scenario_id}?kind=pre|post&locale=`
Возвращает вопросы **без правильных ответов**.

```jsonc
// 200 OK
{
  "scenario_id": "consumer_dispute_001",
  "kind": "pre",
  "questions": [
    { "question_id": "q1",
      "text": "Покупатель обнаружил недостаток товара в период гарантии. Кто, как правило, должен доказать, что недостаток возник по вине покупателя?",
      "options": [
        { "id": "a", "text": "Покупатель" },
        { "id": "b", "text": "Продавец" },
        { "id": "c", "text": "Суд сам устанавливает это без доказательств" }
      ] },
    { "question_id": "q2",
      "text": "Какой документ подтверждает факт и дату покупки?",
      "options": [ { "id": "a", "text": "Кассовый чек" }, { "id": "b", "text": "Рекламный буклет" }, { "id": "c", "text": "Отзыв в интернете" } ] }
  ],
  "total": 10
}
```
Тексты вопросов и правильные ответы утверждает Участник 4 со ссылками на статьи. Примеры выше показывают только формат.

#### `POST /api/quiz/{scenario_id}/attempts`
Тело: `{ "user_id", "kind": "pre", "answers": { "q1": "b", ... } }`
1. Проверить ответы по `quiz.json`.
2. Сохранить в `quiz_attempts`.
3. Для `post`: найти последнюю попытку `pre` и посчитать прирост.
```jsonc
// 201 Created (kind = post)
{
  "attempt_id": 57,
  "score": 8, "max": 10,
  "review": [
    { "question_id": "q1", "chosen_option": "a", "correct": false, "correct_option": "b",
      "explanation": "В гарантийный период обязанность доказать, что недостаток возник по вине покупателя, обычно лежит на продавце.",
      "citation_ids": ["CPRL.art.X"] },
    { "question_id": "q2", "chosen_option": "a", "correct": true, "correct_option": "a",
      "explanation": "Чек подтверждает факт и дату покупки.", "citation_ids": ["CPRL.art.Y"] }
  ],
  "learning_gain": { "pre": 4, "post": 8, "max": 10, "delta": 4,
                     "note": "Предварительный показатель, не доказанный эффект обучения" }   // только для post
}
```

### 5.5 Игровые сессии (ядро)

#### `POST /api/sessions`
Тело: `{ "user_id", "scenario_id", "locale" }`
1. Проверить, что сценарий существует.
2. Создать запись в `sessions` (с `scenario_version`, `locale`, `started_at`).
3. `state = engine.initial_state(scenario)`, этап `INTRO`.
4. Записать событие аналитики `session_started`.
5. Вернуть `GameState`. В `events` — вступительные реплики.

```jsonc
// 201 Created
{
  "session_id": "a3e1b0c4-5d2f-4c8e-8f71-2b9d6e0a1c33",
  "scenario_id": "consumer_dispute_001",
  "locale": "ru",
  "stage": "INTRO",
  "stage_title": "Введение",
  "stage_index": 1, "stage_count": 9,
  "score": 0, "max_score": 100,
  "evidence": [],
  "available_actions": [
    { "action_id": "enter_courtroom", "kind": "continue", "label": "Войти в зал суда",
      "target_character": null, "evidence_id": null, "options": null, "hint_citation_ids": [] }
  ],
  "events": [
    { "type": "say", "speaker": "narrator", "text": "Учебное дело. Март 2026 года. Покупатель обратился в суд с иском к магазину электроники.", "emotion": "neutral" },
    { "type": "say", "speaker": "plaintiff", "text": "Телефон перестал заряжаться через 10 дней, а в магазине мне сказали, что я сам его сломал.", "emotion": "worried" }
  ],
  "finished": false,
  "legal_date": "2026-03-01"
}
```

#### `GET /api/sessions/{session_id}?locale=`
Нужен для перезагрузки страницы и смены языка.
1. Загрузить сессию и лог действий.
2. `state = engine.replay(scenario, actions)`.
3. Вернуть `GameState`, где `events` — события **последнего** действия, чтобы восстановить текущую реплику.

```jsonc
// 200 OK (игрок перезагрузил страницу на этапе исследования доказательств)
{
  "session_id": "a3e1b0c4-5d2f-4c8e-8f71-2b9d6e0a1c33",
  "scenario_id": "consumer_dispute_001",
  "locale": "ru",
  "stage": "EVIDENCE_EXAMINATION",
  "stage_title": "Исследование доказательств",
  "stage_index": 6, "stage_count": 9,
  "score": 30, "max_score": 100,
  "evidence": [
    { "id": "contract", "title": "Договор купли-продажи", "type": "document", "reviewed": true },
    { "id": "receipt",  "title": "Кассовый чек",          "type": "document", "reviewed": true },
    { "id": "warranty", "title": "Гарантийный талон",     "type": "document", "reviewed": false },
    { "id": "chat",     "title": "Переписка с продавцом", "type": "document", "reviewed": true }
  ],
  "available_actions": [
    { "action_id": "present_receipt",  "kind": "evidence", "label": "Представить кассовый чек",
      "target_character": "judge", "evidence_id": "receipt", "options": null, "hint_citation_ids": ["CPC.art.Z"] },
    { "action_id": "present_chat",     "kind": "evidence", "label": "Представить переписку с продавцом",
      "target_character": "judge", "evidence_id": "chat", "options": null, "hint_citation_ids": ["CPC.art.Z"] },
    { "action_id": "ask_defendant_defect_cause", "kind": "question", "label": "На чём основан вывод, что телефон сломал покупатель?",
      "target_character": "defendant_rep", "evidence_id": null, "options": null, "hint_citation_ids": [] },
    { "action_id": "to_motions", "kind": "continue", "label": "Перейти к ходатайствам",
      "target_character": null, "evidence_id": null, "options": null, "hint_citation_ids": [] }
  ],
  "events": [
    { "type": "say", "speaker": "judge", "text": "Переходим к исследованию доказательств. Сторона истца, что вы представляете суду?", "emotion": "neutral" }
  ],
  "finished": false,
  "legal_date": "2026-03-01"
}
```

#### `PATCH /api/sessions/{session_id}`
Тело: `{ "locale": "uz-Latn" }`. Меняет язык сессии. Следующие ответы приходят на новом языке, состояние игры не меняется.
1. Проверить, что язык поддерживается → иначе `422`.
2. Обновить `sessions.locale`.
3. Вернуть то же состояние, переведённое на новый язык.

```jsonc
// 200 OK (то же состояние, что выше, но на uz-Latn; сокращено)
{
  "session_id": "a3e1b0c4-5d2f-4c8e-8f71-2b9d6e0a1c33",
  "locale": "uz-Latn",
  "stage": "EVIDENCE_EXAMINATION",
  "stage_title": "Dalillarni tekshirish",
  "stage_index": 6, "stage_count": 9,
  "score": 30, "max_score": 100,
  "evidence": [ { "id": "receipt", "title": "Kassa cheki", "type": "document", "reviewed": true } ],
  "available_actions": [
    { "action_id": "present_receipt", "kind": "evidence", "label": "Kassa chekini taqdim etish",
      "target_character": "judge", "evidence_id": "receipt", "options": null, "hint_citation_ids": ["CPC.art.Z"] }
  ],
  "events": [ { "type": "say", "speaker": "judge", "text": "<реплика судьи на uz-Latn из texts.uz-Latn.json>", "emotion": "neutral" } ],
  "finished": false,
  "legal_date": "2026-03-01"
}
```
Узбекские тексты в примерах показывают только формат. Финальные формулировки и термины утверждает Участник 4 по глоссарию.

#### `POST /api/sessions/{session_id}/actions`
Тело: `{ "action_id": "present_receipt", "params": { "option": "refund" } }` (`params` только для `decision`).
1. Загрузить сессию. Если она завершена → `409 SESSION_FINISHED`.
2. Восстановить состояние из лога (`replay`).
3. `engine.apply(scenario, state, action_id, params)`:
   - проверить условие `requires` → иначе `409 ILLEGAL_ACTION`;
   - применить эффекты (`set`, `unset`, `score`, `say`, `react`, `add_evidence`, `mistake`, `legal_note`);
   - проверить переходы этапа (первый подходящий срабатывает);
   - на этапе `DECISION` после финального выбора вызвать `evaluate()` и перейти в `RESULTS`.
4. В одной транзакции: добавить строку в `session_actions` (`seq` = следующий номер), записать событие `action` в `events`. Если дело закончено, заполнить `sessions.finished_at` и `outcome`.
5. Перевести тексты событий и действий на язык сессии.
6. Вернуть `GameState`.

Пример 1: обычное действие `{ "action_id": "present_receipt" }`
```jsonc
// 200 OK
{
  "session_id": "a3e1b0c4-5d2f-4c8e-8f71-2b9d6e0a1c33",
  "scenario_id": "consumer_dispute_001",
  "locale": "ru",
  "stage": "EVIDENCE_EXAMINATION",
  "stage_title": "Исследование доказательств",
  "stage_index": 6, "stage_count": 9,
  "score": 40, "max_score": 100,
  "evidence": [
    { "id": "contract", "title": "Договор купли-продажи", "type": "document", "reviewed": true },
    { "id": "receipt",  "title": "Кассовый чек",          "type": "document", "reviewed": true, "presented": true },
    { "id": "warranty", "title": "Гарантийный талон",     "type": "document", "reviewed": false },
    { "id": "chat",     "title": "Переписка с продавцом", "type": "document", "reviewed": true }
  ],
  "available_actions": [
    { "action_id": "present_chat", "kind": "evidence", "label": "Представить переписку с продавцом",
      "target_character": "judge", "evidence_id": "chat", "options": null, "hint_citation_ids": ["CPC.art.Z"] },
    { "action_id": "ask_defendant_defect_cause", "kind": "question", "label": "На чём основан вывод, что телефон сломал покупатель?",
      "target_character": "defendant_rep", "evidence_id": null, "options": null, "hint_citation_ids": [] },
    { "action_id": "to_motions", "kind": "continue", "label": "Перейти к ходатайствам",
      "target_character": null, "evidence_id": null, "options": null, "hint_citation_ids": [] }
  ],
  "events": [
    { "type": "say",   "speaker": "plaintiff_rep", "text": "Уважаемый суд, представляю кассовый чек, который подтверждает дату и место покупки.", "emotion": "confident" },
    { "type": "react", "character": "judge", "reaction": "nod" },
    { "type": "say",   "speaker": "judge", "text": "Суд принимает чек к рассмотрению.", "emotion": "neutral" },
    { "type": "score", "delta": 10, "total": 40 },
    { "type": "legal_note", "citation_id": "CPC.art.Z", "text": "Почему это важно: сторона подтверждает обстоятельства, на которые ссылается, доказательствами." }
  ],
  "finished": false,
  "legal_date": "2026-03-01"
}
```

Пример 2: финальное решение `{ "action_id": "final_claim", "params": { "option": "refund" } }`
```jsonc
// 200 OK
{
  "session_id": "a3e1b0c4-5d2f-4c8e-8f71-2b9d6e0a1c33",
  "scenario_id": "consumer_dispute_001",
  "locale": "ru",
  "stage": "RESULTS",
  "stage_title": "Решение",
  "stage_index": 9, "stage_count": 9,
  "score": 70, "max_score": 100,
  "evidence": [ "...как раньше..." ],
  "available_actions": [],
  "events": [
    { "type": "say", "speaker": "plaintiff_rep", "text": "Просим расторгнуть договор и взыскать уплаченную сумму.", "emotion": "confident" },
    { "type": "say", "speaker": "judge", "text": "Суд удаляется для вынесения решения.", "emotion": "neutral" },
    { "type": "react", "character": "judge", "reaction": "gavel" },
    { "type": "stage_changed", "stage": "RESULTS", "title": "Решение" }
  ],
  "finished": true,
  "legal_date": "2026-03-01"
}
```
После `finished: true` фронт переходит на экран результатов и вызывает `GET /result`.

Пример 3: недоступное действие
```jsonc
// 409 Conflict
{ "error": { "code": "ILLEGAL_ACTION",
             "message": "Действие недоступно на этом этапе",
             "details": { "action_id": "final_claim", "stage": "EVIDENCE_EXAMINATION" } } }
```

Гарантия: одинаковый лог действий всегда даёт одинаковое состояние и результат. LLM в этом эндпоинте **не вызывается**.

#### `GET /api/sessions/{session_id}/result?locale=`
Если игра не завершена → `409 SESSION_NOT_FINISHED`.
1. `replay` → `evaluate(scenario, state)`.
2. Собрать правильный путь: для каждой точки решения — что выбрал игрок, что было правильно, объяснение и статьи.
3. Достать `CitationView` для всех упомянутых `citation_id` на дату `legal_date`.
4. Посчитать время прохождения и долю правильных решений.
```jsonc
// 200 OK
{
  "session_id": "a3e1b0c4-5d2f-4c8e-8f71-2b9d6e0a1c33",
  "scenario_id": "consumer_dispute_001",
  "outcome": "claim_partially_granted",
  "outcome_title": "Иск удовлетворён частично",
  "ruling_text": "Учебное решение: суд признал, что недостаток выявлен в гарантийный период, но истец не заявил ходатайство о проведении экспертизы, поэтому причина недостатка осталась спорной.",
  "score": 70, "max_score": 100,
  "correct_decision_rate": 0.67,
  "time_spent_sec": 742,
  "decisions": [
    { "decision_id": "present_evidence", "stage": "EVIDENCE_EXAMINATION", "chosen": "receipt_and_chat", "chosen_label": "Чек и переписка",
      "best": "receipt_and_chat", "best_label": "Чек и переписка", "correct": true,
      "explanation": "Вы подтвердили факт покупки и то, что покупатель обращался к продавцу.", "citation_ids": ["CPC.art.Z"] },
    { "decision_id": "motion_expertise", "stage": "MOTIONS", "chosen": "none", "chosen_label": "Ходатайство не заявлено",
      "best": "request_expertise", "best_label": "Ходатайство о назначении экспертизы", "correct": false,
      "explanation": "Причина недостатка была главным спорным вопросом. Экспертиза могла его разрешить.", "citation_ids": ["CPC.art.W"] },
    { "decision_id": "final_claim", "stage": "DECISION", "chosen": "refund", "chosen_label": "Возврат уплаченной суммы",
      "best": "refund", "best_label": "Возврат уплаченной суммы", "correct": true,
      "explanation": "Требование соответствует правам покупателя при недостатке товара.", "citation_ids": ["CPRL.art.X"] }
  ],
  "mistakes": [
    { "category": "procedure", "title": "Не заявлено ходатайство об экспертизе",
      "explanation": "Без экспертизы суд не смог установить причину поломки.",
      "correct_action": "На этапе ходатайств попросить суд назначить экспертизу товара.",
      "citation_ids": ["CPC.art.W"] }
  ],
  "citations": [
    { "citation_id": "CPRL.art.X",
      "document_title": "Закон Республики Узбекистан «О защите прав потребителей»", "document_type": "law", "document_number": "<с lex.uz>",
      "article_number": "X", "article_title": "<название статьи из базы>", "part": null,
      "text": "<дословный текст статьи из базы>",
      "language": "ru", "is_official_text": true,
      "edition": { "effective_from": "<дата редакции>", "effective_to": null, "status": "current" },
      "source_url": "https://lex.uz/docs/<id>", "last_verified_at": "2026-11-21", "translation_note": null }
  ],
  "recommendations": [
    { "topic": "Доказательства и ходатайства в гражданском процессе", "citation_ids": ["CPC.art.W"], "scenario_id": null }
  ],
  "disclaimer": "Это учебный результат. Реальный суд оценивает все обстоятельства конкретного дела, и его решение может отличаться."
}
```

### 5.6 NPC (P2)

#### `POST /api/sessions/{session_id}/npc/{character_id}/ask`
Тело: `{ "text": "Когда вы обнаружили недостаток?" }`
1. Восстановить состояние. Взять список вопросов, которые разрешено задавать этому персонажу на текущем этапе.
2. LLM (быстрая модель) классифицирует текст в один `question_id` из списка или `unknown`. Ответ строго из enum.
3. Движок возвращает `fact_ids` для этого вопроса. Это и есть источник истины.
4. LLM перефразирует факты голосом персонажа на нужном языке. Промпт запрещает добавлять факты.
5. Валидация: ответ не пустой и не длиннее лимита. Если что-то не так, используется заготовленная реплика.
6. Таймаут 4 секунды → заготовленная реплика.
7. Если вопрос из списка ключевых, можно поставить флаг через обычное игровое действие `ask_<question_id>`, а не из ответа LLM.
```jsonc
// 200 OK (ответ сгенерирован LLM на основе факта сценария)
{ "speaker": "plaintiff", "text": "Я заметил, что телефон не заряжается, на десятый день после покупки, и в тот же день написал продавцу.",
  "question_id": "q_defect_date", "fact_ids": ["F_DEFECT_DATE", "F_COMPLAINT_SENT"], "ai_generated": true }

// 200 OK (LLM недоступен или вопрос не распознан → заготовленная реплика)
{ "speaker": "plaintiff", "text": "Извините, я не понял вопрос. Спросите, пожалуйста, иначе.",
  "question_id": "unknown", "fact_ids": [], "ai_generated": false }
```

### 5.7 Правовая база

#### `GET /api/legal/citations/{citation_id}?date=&locale=`
1. По `citations` найти документ, номер статьи и часть.
2. Выбрать редакцию: `effective_from <= date < effective_to` (по умолчанию `date` = сегодня).
3. Язык: нужный `locale`; если его нет — официальный текст плюс `translation_note`.
4. Если редакция на эту дату не подтверждена → `edition.status = "unconfirmed"`.
5. Вернуть `CitationView`.

```jsonc
// 200 OK  GET /api/legal/citations/CPRL.art.X?date=2026-03-01&locale=ru
{
  "citation_id": "CPRL.art.X",
  "article_id": 412,
  "document_title": "Закон Республики Узбекистан «О защите прав потребителей»",
  "document_type": "law",
  "document_number": "<с lex.uz>",
  "article_number": "X",
  "article_title": "<название статьи из базы>",
  "part": null,
  "parts": [
    { "part_number": 1, "kind": "text", "text": "<дословный текст части 1>" },
    { "part_number": 2, "kind": "text", "text": "<дословный текст части 2>" }
  ],
  "text": "<дословный текст всей статьи>",
  "language": "ru",
  "is_official_text": true,
  "edition": { "effective_from": "<дата редакции>", "effective_to": null, "status": "current" },
  "source_url": "https://lex.uz/docs/<id>",
  "last_verified_at": "2026-11-21",
  "translation_note": null
}

// 200 OK  редакция на дату не подтверждена, текст на нужном языке отсутствует
{
  "citation_id": "CPRL.art.X",
  "...": "...те же поля...",
  "language": "uz-Latn",
  "is_official_text": null,
  "edition": { "effective_from": null, "effective_to": null, "status": "unconfirmed" },
  "translation_note": "Редакция на 01.03.2026 не подтверждена. Показан текст действующей редакции."
}

// 404 Not Found
{ "error": { "code": "NOT_FOUND", "message": "Статья не найдена", "details": { "citation_id": "CPRL.art.999" } } }
```

#### `GET /api/legal/articles/{article_id}`
То же самое, но по прямому id статьи. Нужен для ссылок из результатов поиска.
1. Найти статью по `id` → иначе `404`.
2. Подтянуть редакцию и документ.
3. Если для статьи есть `citation_id`, вернуть и его.

```jsonc
// 200 OK  GET /api/legal/articles/412
{
  "article_id": 412,
  "citation_id": "CPRL.art.X",          // null, если статья не используется в сценариях
  "document_title": "Закон Республики Узбекистан «О защите прав потребителей»",
  "document_type": "law",
  "article_number": "X",
  "article_title": "<название статьи из базы>",
  "chapter": "<номер главы>",
  "parts": [ { "part_number": 1, "kind": "text", "text": "<дословный текст>" } ],
  "text": "<дословный текст всей статьи>",
  "language": "ru",
  "is_official_text": true,
  "edition": { "effective_from": "<дата редакции>", "effective_to": null, "status": "current" },
  "source_url": "https://lex.uz/docs/<id>",
  "last_verified_at": "2026-11-21",
  "translation_note": null
}
```

#### `GET /api/legal/search?q=&locale=&date=&limit=10`
Гибридный поиск (раздел 6) без LLM. Используется в юридической справке и для отладки RAG.
```jsonc
// 200 OK  GET /api/legal/search?q=вернуть телефон с недостатком&locale=ru&limit=10
{
  "query": "вернуть телефон с недостатком",
  "detected_language": "ru",
  "date": "2026-11-21",
  "results": [
    { "chunk_id": 1187, "article_id": 412, "citation_id": "CPRL.art.X",
      "document_title": "Закон Республики Узбекистан «О защите прав потребителей»",
      "article_number": "X", "article_title": "<название статьи из базы>", "parts": "1–2",
      "snippet": "…<фрагмент дословного текста с совпадением>…",
      "edition_status": "current", "source_url": "https://lex.uz/docs/<id>",
      "score": 0.0325,
      "match": { "exact_article": false, "dense_rank": 1, "sparse_rank": 2 } },
    { "chunk_id": 2240, "article_id": 905, "citation_id": null,
      "document_title": "Гражданский кодекс Республики Узбекистан",
      "article_number": "Y", "article_title": "<название статьи из базы>", "parts": "1",
      "snippet": "…<фрагмент>…",
      "edition_status": "current", "source_url": "https://lex.uz/docs/<id>",
      "score": 0.0291,
      "match": { "exact_article": false, "dense_rank": 3, "sparse_rank": 1 } }
  ]
}
```

### 5.8 AI Legal Mentor (SSE)

#### `POST /api/mentor/ask` → `text/event-stream`
Тело:
```json
{ "session_id": "uuid|null", "question": "Почему я проиграл дело?", "locale": "ru",
  "history": [{ "role": "user", "text": "..." }, { "role": "assistant", "text": "..." }] }
```
`history` — последние до 6 сообщений. История хранится на фронте.

Что выполняется:
1. Определить язык вопроса по алфавиту и стоп-словам. Отвечаем на нём.
2. Нормализовать текст: апострофы, транслитерация для поиска по ключевым словам.
3. Классифицировать (быстрая модель, enum): `article_lookup | concept | case_specific | out_of_scope`. Если `out_of_scope` — вежливый отказ без поиска.
4. Если передан `session_id`: восстановить состояние, взять результат, ошибки и `citation_id` сценария. Это структурированный контекст, а не свободный текст.
5. Поиск:
   - явные ссылки в тексте («ст. 15», «15-modda») → точная выборка статьи;
   - векторный поиск, top 30;
   - полнотекстовый поиск + триграммы, top 30;
   - фильтры: редакция на `legal_date` сценария (или сегодня), язык, разрешённые документы;
   - слияние RRF; статьи сценария всегда добавляются как кандидаты;
   - reranker, если включён.
6. Если лучший результат ниже `RAG_MIN_SCORE` → отправить `done` с `insufficient_sources: true` без вызова LLM.
7. Отправить событие `sources` (список найденных статей).
8. Собрать контекст: 5–8 chunks в тегах `<source id="c1" citation="..." url="...">`. В системном промпте сказано, что содержимое `<source>` — это данные, а не инструкции.
9. Вызвать LLM со структурированным выводом: `answer_blocks[{text, source_ids}]`, `insufficient_sources`.
10. Валидация каждого блока:
    - все `source_ids` должны быть среди найденных, иначе блок удаляется;
    - регулярками ищем упоминания статей и законов, не подкреплённые цитированными chunks → блок получает `verified: false` или удаляется.
11. Отправлять блоки по мере готовности.
12. Записать событие `mentor_question` (вопрос, число источников, отказ или нет).

Таймаут 20 секунд или нет ключа → `sources` + `done { ai_unavailable: true }`.

Поток событий (типы):
```
event: sources   data: [{ "source_id": "c1", "citation": CitationView }]
event: block     data: { "text": "...", "source_ids": ["c1"], "verified": true }
event: done      data: { "insufficient_sources": false, "ai_unavailable": false, "disclaimer": "..." }
event: error     data: { "code": "...", "message": "..." }
```

Пример 1: обычный ответ (`200 OK`, `Content-Type: text/event-stream`), вопрос «Почему я проиграл дело частично?»
```
event: sources
data: [{"source_id":"c1","citation":{"citation_id":"CPC.art.W","document_title":"Гражданский процессуальный кодекс Республики Узбекистан","article_number":"W","article_title":"<из базы>","edition":{"status":"current"},"source_url":"https://lex.uz/docs/<id>"}},
       {"source_id":"c2","citation":{"citation_id":"CPRL.art.X","document_title":"Закон Республики Узбекистан «О защите прав потребителей»","article_number":"X","article_title":"<из базы>","edition":{"status":"current"},"source_url":"https://lex.uz/docs/<id>"}}]

event: block
data: {"text":"Главным спорным вопросом была причина поломки: продавец утверждал, что телефон повредил покупатель.","source_ids":[],"verified":true}

event: block
data: {"text":"Чтобы установить причину недостатка, сторона может попросить суд назначить экспертизу. Вы не заявили это ходатайство, поэтому вопрос остался без ответа.","source_ids":["c1"],"verified":true}

event: block
data: {"text":"При этом само требование о возврате денег вы выбрали правильно: закон даёт покупателю такое право при недостатке товара.","source_ids":["c2"],"verified":true}

event: done
data: {"insufficient_sources":false,"ai_unavailable":false,"disclaimer":"Образовательное объяснение, не юридическая консультация"}
```
Блок без юридического утверждения может иметь пустой `source_ids`. Блок с юридическим утверждением без источника валидатор удаляет или помечает `"verified": false`.

Пример 2: источников недостаточно (вопрос «Как оформить развод?»)
```
event: sources
data: []

event: done
data: {"insufficient_sources":true,"ai_unavailable":false,"message":"В загруженной правовой базе недостаточно источников, чтобы ответить на этот вопрос.","disclaimer":"Образовательное объяснение, не юридическая консультация"}
```

Пример 3: нет ключа LLM или таймаут
```
event: sources
data: [{"source_id":"c1","citation":{"citation_id":"CPC.art.W","...":"..."}}]

event: done
data: {"insufficient_sources":false,"ai_unavailable":true,"message":"AI-объяснение сейчас недоступно. Вот нормы, которые относятся к вашему вопросу.","disclaimer":"Образовательное объяснение, не юридическая консультация"}
```

Пример 4: ошибка
```
event: error
data: {"code":"NOT_FOUND","message":"Сессия не найдена"}
```

### 5.9 Аналитика

#### `POST /api/events` (P1)
Тело: `{ "user_id", "session_id?", "events": [{ "type": "legal_panel_opened", "payload": {...}, "ts": "..." }] }`. Клиентские события пишутся пачкой в `events`. Разрешённые типы: `legal_panel_opened`, `evidence_opened`, `mentor_opened`, `screen_view`.
1. Отбросить события неизвестного типа (без ошибки, они просто считаются в `rejected`).
2. Ограничение: не больше 50 событий за запрос → иначе `422`.
3. Вставить одним `INSERT`.

```jsonc
// 202 Accepted
{ "accepted": 3, "rejected": 0 }
```

#### `GET /api/analytics/scenarios/{scenario_id}/summary` (P1, для презентации и будущих преподавателей)
Агрегаты через SQL-представления:
```jsonc
// 200 OK
{ "sessions_started": 31, "completion_rate": 0.84, "avg_score": 68, "correct_decision_rate": 0.61,
  "avg_time_sec": 690, "avg_learning_gain": 2.9,
  "mistakes_by_category": { "evidence": 18, "procedure": 7, "substantive_law": 11, "remedy_choice": 9 },
  "note": "Предварительные данные, малая выборка" }
```

---

## 6. Игровой движок (`game_engine/`)

Чистые функции, без БД, сети и времени (время передаётся параметром).

```python
initial_state(scenario) -> GameState
available_actions(scenario, state) -> list[Action]
apply(scenario, state, action_id, params=None) -> tuple[GameState, list[Event]]   # raises IllegalAction
replay(scenario, actions: list[LoggedAction]) -> tuple[GameState, list[Event]]   # события последнего шага
evaluate(scenario, state) -> Outcome
```

- Условия задаются в JSON: `all`, `any`, `not`, `flag`, `gte`, `always`. Интерпретатор около 50 строк.
- Порядок проверки `evaluation_rules` фиксирован: первое подходящее правило определяет исход. Последнее правило всегда `always: true`.
- Эффекты: `set`, `unset`, `score`, `say`, `react`, `add_evidence`, `mistake`, `legal_note`, `goto`.
- Этапы FSM: `INTRO → CASE_REVIEW → PREPARATION → HEARING_OPENING → PARTY_EXPLANATIONS → EVIDENCE_EXAMINATION ⇄ MOTIONS → DEBATE → DECISION → RESULTS`. Названия и порядок этапов проверяет Участник 4 по ГПК.

---

## 7. Модель данных

```
users(id uuid pk, display_name, group_id null, created_at)
sessions(id uuid pk, user_id fk, scenario_id, scenario_version, locale,
         started_at, finished_at null, outcome null, score null)
session_actions(session_id fk, seq int, action_id, params jsonb, ts,  pk(session_id, seq))
events(id bigserial, session_id null, user_id, type, payload jsonb, ts)
quiz_attempts(id, user_id, scenario_id, kind, answers jsonb, score, max_score, ts)
-- правовые таблицы: см. INGESTION.md, раздел 6
```

SQL-представления: `v_session_metrics`, `v_learning_gain`, `v_mistakes`.

---

## 8. Что происходит при старте сервера

1. Миграции Alembic (`alembic upgrade head`).
2. Загрузка всех `scenarios/*/scenario.json` и проверка схемы Pydantic.
3. Проверки (сервер **не стартует**, если хоть одна не проходит):
   - каждый `citation_id` находится в таблице `citations` и у него есть статья;
   - каждый `text_key` переведён на ru и uz-Latn (для uz-Cyrl — предупреждение);
   - каждый этап достижим из `INTRO`, а из каждого этапа можно попасть в `RESULTS`;
   - каждый `evaluation_rules` заканчивается правилом `always`.
4. Прогрев: загрузка модели эмбеддингов (если локальная) и кэш сценариев.

---

## 9. Тесты (обязательные)

| Тест | Что проверяет |
|---|---|
| `test_transitions` | каждый этап достижим; недоступные действия дают `IllegalAction` |
| `test_golden_paths` | 3 эталонных прохождения (выигрыш, частично, проигрыш) дают ожидаемый исход и баллы |
| `test_determinism` | `replay` одного лога 100 раз даёт одно и то же |
| `test_random_walk` | 500 случайных допустимых последовательностей без падений, всегда доходят до `RESULTS` |
| `test_api_flow` | создать сессию → пройти эталонный путь через API → получить результат |
| `test_citations_resolve` | все ссылки сценария находятся в базе |
| `test_mentor_no_ai` | без ключа наставник отдаёт `sources` + `ai_unavailable` |
| `test_mentor_validation` | блок с выдуманным `source_id` или номером статьи удаляется |
| `test_locales` | все ответы есть на ru и uz-Latn |

---

## 10. Критерии готовности

- [ ] Все эндпоинты из раздела 5 (кроме P2) есть в `/docs` и покрыты тестом хотя бы одного сценария.
- [ ] Эталонное прохождение через API работает без `ANTHROPIC_API_KEY`.
- [ ] Повтор того же лога действий даёт тот же результат.
- [ ] Наставник никогда не показывает статью, которой нет в базе.
- [ ] `docker compose up` поднимает сервер с готовой базой без ручных шагов.
