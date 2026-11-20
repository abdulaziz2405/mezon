# ADOLAT — Legal Ingestion: спецификация

Владелец: Участник 2 (код пайплайна). Участник 4: получение документов, метаданные, юридическая проверка.
Связанные документы: [ARCHITECTURE.md](../ARCHITECTURE.md), [BACKEND.md](BACKEND.md).

---

## 1. Задача

Превратить официальные тексты законодательства Узбекистана в базу, где:
- каждая статья хранится **дословно** и связана с документом, редакцией, языком и ссылкой на lex.uz;
- по статьям работает гибридный поиск (векторы + полнотекстовый);
- сценарии ссылаются на статьи через стабильные `citation_id`, и каждая ссылка проверяется.

Ingestion — это **офлайн-инструмент** (CLI), а не часть API. Он запускается вручную или в CI. Результат — заполненная база и seed-файл для демо.

---

## 2. Можно ли это автоматизировать?

Коротко: **всё, кроме получения документов и юридической проверки, — да, полностью**.

| Шаг | Автоматизация | Почему |
|---|---|---|
| Получение документов с lex.uz | **Вручную** (в MVP) | Открытого API у lex.uz мы не подтвердили. Массовый скрапинг без разрешения запрещён брифом. Сначала читаем условия использования lex.uz и robots.txt, результат записываем в `sources.yaml` |
| Скачивание по готовому списку URL | **Полуавтоматически**, только если условия lex.uz это разрешают | Команда `fetch` скачивает только документы из `sources.yaml`, по одному, с паузой, с понятным User-Agent, соблюдая robots.txt. Без обхода сайта, CAPTCHA и авторизации. Если условия не позволяют, команда выключена (`FETCH_ENABLED=false`) |
| Парсинг, нормализация, разбиение на статьи | **Полностью** | Детерминированный код на регулярных выражениях |
| Версионирование, дедупликация | **Полностью** | Хэш содержимого |
| Chunking, эмбеддинги, загрузка в БД | **Полностью** | |
| Проверки целостности | **Полностью** | Команда `check`, запускается и в CI |
| Обнаружение изменений в законе | **Полуавтоматически** | `diff` показывает, какие статьи изменились между редакциями. Новая редакция загружается со статусом `pending_review` и не используется, пока человек её не подтвердит |
| Метаданные (даты редакций, официальность текста) | **Вручную** | Это юридические факты. LLM их не угадывает |
| Привязка статей к сценарию (`citations.yaml`) | **Вручную**, с помощью поиска | Участник 4 ищет нужные нормы через `/api/legal/search`, но решение о том, какая статья применима, принимает человек |
| Оценка качества поиска (Recall@K) | **Полностью** | Команда `eval` на заранее подготовленном наборе вопросов |

Итог для MVP: **одна команда** `python -m legal_ingestion run` превращает папку `raw/` и `sources.yaml` в готовую базу. Люди делают только то, что требует юридического решения.

На будущее (roadmap, хорошо звучит для бизнес-жюри): официальное соглашение о доступе к данным с оператором lex.uz. Тогда можно будет заменить `fetch` на регулярную синхронизацию.

---

## 3. Входные данные

### 3.1 Папка `legal_data/raw/`

Файлы ровно в том виде, в котором их скачали с lex.uz (HTML-страница, DOCX или PDF, если сайт их отдаёт). **Никогда не редактируются руками.** Имя файла: `<document_key>_<language>_<edition_date>.<ext>`, например `CPRL_ru_2024-01-01.html`.

### 3.2 `legal_data/sources.yaml`

Заполняет Участник 4.

```yaml
terms_of_use:
  checked_at: 2026-11-21
  checked_by: "<инициалы>"
  summary: "<что сказано в условиях lex.uz про копирование и автоматический доступ>"
  fetch_allowed: false          # включает или выключает команду fetch

documents:
  - document_key: CPRL                       # короткий ключ, используется в citation_id
    title:
      ru: "Закон Республики Узбекистан «О защите прав потребителей»"
      uz-Latn: "<официальное название>"
    document_type: law                       # constitution | code | law | plenum_resolution | decree
    document_number: "<с lex.uz>"
    adoption_date: "<с lex.uz>"
    issuing_body: "<с lex.uz>"
    source_url: "https://lex.uz/docs/<id>"
    status: in_force                         # in_force | repealed
    versions:
      - language: ru
        file: raw/CPRL_ru_<дата>.html
        source_url: "https://lex.uz/docs/<id>?<параметры редакции/языка, если есть>"
        effective_from: "<дата редакции, как указана на lex.uz>"
        effective_to: null
        is_official_text: null               # true/false ставит Legal QA, null = не проверено
        retrieved_at: 2026-11-21
        verified_by: null                    # пока null, редакция не активна
```

### 3.3 `legal_data/citations.yaml`

Стабильные идентификаторы, на которые ссылаются сценарии. Заполняет Участник 4 после проверки.

```yaml
- citation_id: CPRL.art.X          # X — реальный номер статьи после проверки
  document_key: CPRL
  article_number: "X"
  part: null
  topic: "Права потребителя при продаже товара ненадлежащего качества"
  verified_by: "<инициалы>"
```

### 3.4 Минимальный корпус для сценария

1. Конституция Республики Узбекистан (права граждан, судебная защита).
2. Гражданский кодекс: купля-продажа, качество товара, гарантия.
3. Закон «О защите прав потребителей».
4. Гражданский процессуальный кодекс: стадии разбирательства, доказательства, бремя доказывания, ходатайства.
5. Постановления Пленума Верховного суда по делам о защите прав потребителей, если они найдены на официальных ресурсах.

Конкретные главы и статьи определяет Участник 4. В коде и сценариях **нет номеров статей, пока они не проверены по загруженному тексту**.

---

## 4. Пайплайн

```
sources.yaml + raw/
   │
   ├─1─ validate   проверка sources.yaml: все файлы на месте, обязательные поля заполнены
   ├─2─ parse      HTML/DOCX/PDF → упорядоченные текстовые блоки
   ├─3─ normalize  дословный текст + текст для поиска
   ├─4─ segment    разделы → главы → статьи → части → пункты
   ├─5─ version    хэш, сравнение с прошлой редакцией, effective_from/to
   ├─6─ chunk      статьи → chunks с заголовком
   ├─7─ embed      векторы (с кэшем)
   ├─8─ load       upsert в Postgres
   ├─9─ link       citations.yaml → таблица citations
   └─10─ check     проверки целостности → отчёт
```

Каждый шаг пишет результат в `legal_data/processed/` (JSONL). Повторный запуск без изменений ничего не меняет (идемпотентность).

### Шаг 2. Parse (`parse.py`)

- HTML → `selectolax` (или BeautifulSoup). Удаляем навигацию, меню и скрипты, оставляем только тело документа. Селектор тела подбирается на реальной странице lex.uz в первые часы и фиксируется в коде.
- DOCX → `python-docx`, по абзацам.
- PDF → `pdfplumber`, только если другого формата нет (хуже сохраняет структуру).
- Выход: `processed/<key>_<lang>_<date>.blocks.jsonl` — `{ "idx": 0, "text": "...", "style": "heading|paragraph|note" }`.
- Сохраняем выделение (жирный и т.п.), если оно помогает найти заголовки.

### Шаг 3. Normalize (`normalize.py`)

Две версии текста:

| Поле | Что делаем | Для чего |
|---|---|---|
| `content` | только убираем мусор: двойные пробелы, неразрывные пробелы, мягкие переносы, артефакты HTML. Нумерация, примечания, ссылки на другие статьи, оговорки «за исключением…» остаются **без изменений** | показываем пользователю, цитируем |
| `content_normalized` | нижний регистр; все варианты апострофа (`ʻ ’ ‘ ' \``, а также `ʼ`) → один символ; ё → е; для узбекского дополнительно транслитерация в латиницу | полнотекстовый поиск |

Транслитерация Cyrl → Latn для узбекского — по таблице (`ш→sh`, `ч→ch`, `ў→oʻ`, `ғ→gʻ`, `қ→q`, `ҳ→h`, `е→e/ye` в начале слова и т.д.). Это нужно только для поиска, юридический текст не транслитерируется.

### Шаг 4. Segment (`segment.py`)

Заголовки ищутся регулярками. Это стартовые варианты: **на реальных файлах их нужно проверить в первые часы** (особенно номера вставленных статей и заголовки на узбекском).

```python
SECTION = {"ru": r"^Раздел\s+([IVXLC\d]+)",  "uz-Latn": r"^([IVXLC\d]+)[- ]?bo[ʻ']lim", "uz-Cyrl": r"^([IVXLC\d]+)[- ]?бўлим"}
CHAPTER = {"ru": r"^Глава\s+(\d+)",          "uz-Latn": r"^(\d+)[- ]?bob",              "uz-Cyrl": r"^(\d+)[- ]?боб"}
ARTICLE = {
    "ru":      r"^Статья\s+(\d+(?:[-–]\d+|[¹²³⁴⁵⁶⁷⁸⁹]+)?)\.\s*(.*)$",
    "uz-Latn": r"^(\d+(?:[-–]\d+|[¹²³⁴⁵⁶⁷⁸⁹]+)?)[- ]?modda\.\s*(.*)$",
    "uz-Cyrl": r"^(\d+(?:[-–]\d+|[¹²³⁴⁵⁶⁷⁸⁹]+)?)[- ]?модда\.\s*(.*)$",
}
```

Правила:
- Статья — от заголовка до следующего заголовка статьи, главы или раздела.
- Части статьи — абзацы внутри неё, нумеруются по порядку (`part_number` = 1, 2, …).
- Пункты и подпункты (`1)`, `а)`, `—`) **остаются внутри своей части**, не отделяются.
- Примечания (`Примечание.`, `Izoh.`) сохраняются как отдельная часть с `kind = note`.
- Статьи вида «утратила силу» сохраняются с этим текстом (`status = repealed`), чтобы нумерация не ломалась.
- Текст до первой статьи (преамбула) сохраняется как `article_number = "preamble"`.

Выход: `processed/<key>_<lang>_<date>.articles.jsonl`
```json
{ "document_key": "CPRL", "language": "ru", "section": "I", "chapter": "2", "article_number": "X",
  "article_title": "...", "parts": [{ "part_number": 1, "kind": "text", "content": "...", "content_normalized": "..." }],
  "status": "in_force" }
```

### Шаг 5. Version (`version.py`)

- `content_hash` = SHA-256 нормализованного текста всей редакции.
- Хэш совпадает с уже загруженным → редакция пропускается.
- Новая редакция того же документа и языка:
  - загружается со статусом `pending_review`, если `verified_by` пустой;
  - после подтверждения у предыдущей редакции ставится `effective_to` = `effective_from` новой;
  - старая редакция **не удаляется**: она нужна для сценариев с более ранней `legal_date`.
- `diff` между редакциями (по статьям): добавлены, удалены, изменены. Отчёт в `processed/<key>_diff_<old>_<new>.md` для юриста.

### Шаг 6. Chunk (`chunk.py`)

- Одна статья — один chunk, если она не длиннее примерно 800 токенов.
- Длинная статья делится **только по границам частей**, части собираются в группы до 800 токенов. Пункты внутри части не разрываются никогда.
- В начало каждого chunk добавляется заголовок (он попадает и в эмбеддинг, и в контекст LLM):
  ```
  [Закон «О защите прав потребителей» · Статья X. <название> · части 1–2 · редакция от <дата> · ru]
  ```
- Связи: `chunk → article_id → version_id → document_id`.

### Шаг 7. Embed (`embed.py`)

- Интерфейс `Embedder.embed(texts, kind="passage"|"query") -> list[vector]`. Модель выбирается переменной `EMBEDDER`.
- Кандидаты: **BGE-M3** (локально), **multilingual-e5-large** (локально, нужны префиксы `passage:` / `query:`), один облачный мультиязычный API.
- Отправка пачками по 32–64. Кэш по `(model, sha256(text))` в `processed/embeddings_cache/`, чтобы повторный запуск не пересчитывал неизменившееся.
- Размерность вектора берётся из модели и фиксируется в миграции. **Смена модели = пересоздание колонки и полная переиндексация** (`run --reembed`).

### Шаг 8. Load (`load.py`)

Upsert в Postgres в одной транзакции на редакцию. Если что-то падает, редакция не загружается частично.

### Шаг 9. Link (`link.py`)

`citations.yaml` → таблица `citations`. Если статья не найдена, это ошибка, а не предупреждение.

### Шаг 10. Check (`check.py`)

| Проверка | Уровень |
|---|---|
| У каждой редакции есть `source_url`, `retrieved_at`, файл в `raw/` | ошибка |
| Номера статей идут подряд; пропуски перечислены в `sources.yaml` → `known_gaps` | ошибка |
| Нет пустых статей и повторяющихся номеров в одной редакции | ошибка |
| Цепочка chunk → article → version → document не разорвана | ошибка |
| Каждый `citation_id` из `citations.yaml` и из `scenarios/**` находит статью | ошибка |
| У активной редакции заполнен `verified_by` | ошибка |
| `is_official_text = null` | предупреждение |
| Число статей сильно изменилось по сравнению с прошлым запуском | предупреждение |
| Случайные 10 статей выводятся рядом с исходником | ручная проверка |

Отчёт: `processed/check_report.md` (коммитится). Код выхода ≠ 0 при любой ошибке.

---

## 5. Команды CLI

```
python -m legal_ingestion run [--doc CPRL] [--reembed]   # шаги 1–10
python -m legal_ingestion parse|segment|chunk --doc CPRL # отдельный шаг, для отладки
python -m legal_ingestion check                          # только проверки
python -m legal_ingestion diff --doc CPRL                # изменения между редакциями
python -m legal_ingestion activate --doc CPRL --version <id> --by "<инициалы>"   # подтвердить редакцию
python -m legal_ingestion fetch                          # только при fetch_allowed: true
python -m legal_ingestion eval [--k 5]                   # Recall@K на eval/gold.jsonl
python -m legal_ingestion dump                           # legal_data/seed.sql.gz
```

В docker compose есть отдельный сервис `ingest` (профиль `tools`):
```
docker compose --profile tools run --rm ingest run
```

---

## 6. Схема таблиц

```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_trgm;

legal_documents(
  id serial pk, document_key text unique, document_type text, document_number text,
  title jsonb,                       -- {"ru": "...", "uz-Latn": "..."}
  adoption_date date, issuing_body text, source_url text, status text)

document_versions(
  id serial pk, document_id fk, language text,
  is_official_text bool null, effective_from date, effective_to date null,
  review_status text,                -- pending_review | active | superseded
  version_label text, source_url text, raw_file_path text, content_hash text,
  retrieved_at date, verified_by text null, last_verified_at date null,
  unique(document_id, language, content_hash))

articles(
  id serial pk, version_id fk, section text, chapter text,
  article_number text, article_title text, status text,
  content text,                      -- дословно, все части
  content_normalized text,
  unique(version_id, article_number))

article_parts(
  id serial pk, article_id fk, part_number int, kind text,   -- text | note
  content text, content_normalized text)

chunks(
  id serial pk, article_id fk, part_from int, part_to int,
  text text,                         -- с заголовком
  text_normalized text,
  embedding vector(<dim>),
  tsv tsvector GENERATED ALWAYS AS (to_tsvector('simple', text_normalized)) STORED,
  token_count int)

citations(
  citation_id text pk, document_id fk, article_number text, part int null,
  topic text, verified_by text)

CREATE INDEX ON chunks USING gin (tsv);
CREATE INDEX ON articles USING gin (article_title gin_trgm_ops);
-- HNSW-индекс на embedding добавлять, только если точный поиск станет медленным (корпус маленький)
```

---

## 7. Оценка качества поиска

`eval/gold.jsonl` — около 40 вопросов (20 ru, 20 uz-Latn). Пишет Участник 4, проверяет Участник 2.
```json
{ "id": "g01", "lang": "ru", "question": "Можно ли вернуть телефон, если через неделю перестал включаться?",
  "expected_citations": ["CPRL.art.X", "CC.art.Y"], "type": "situation" }
```
Типы вопросов: `exact_ref` («что говорит статья X»), `term` (юридический термин), `situation` (бытовое описание), `out_of_scope` (ответ должен быть отказом).

`eval` считает Recall@5 и Recall@10 отдельно для dense, sparse и hybrid, по каждому языку. Отчёт в `eval/report.md`. По нему выбирается модель эмбеддингов (к 10-му часу), и эти цифры идут в презентацию.

---

## 8. Seed для демо

- `dump` создаёт `legal_data/seed.sql.gz`: все правовые таблицы с уже посчитанными векторами.
- Postgres в docker compose загружает его при первом старте (`/docker-entrypoint-initdb.d/`).
- Значит, на демо не нужны ни скачивание, ни расчёт эмбеддингов, ни API-ключ.
- Seed пересобирается после каждого изменения корпуса и коммитится (если файл большой — Git LFS или release-артефакт).

---

## 9. CI

При изменениях в `legal_data/**`, `scenarios/**` или `backend/legal_ingestion/**`:
1. `parse` + `segment` + `check` (без эмбеддингов, это быстро).
2. Проверка, что все `citation_id` из сценариев находятся.
3. Если запуск ручной, дополнительно `eval` (с моделью, медленно).

---

## 10. План по часам

| Часы | Участник 4 | Участник 2 |
|---|---|---|
| 0–3 | Условия lex.uz → `sources.yaml`; скачать 4–5 актов на ru и uz-Latn | Каркас CLI, схема таблиц, `parse` для HTML |
| 3–6 | Заполнить метаданные редакций | `normalize`, `segment`; проверка регулярок на реальных файлах |
| 6–10 | 40 вопросов в `gold.jsonl` | `chunk`, `embed` для 2–3 моделей, `load`, `eval` → выбор модели |
| 10–14 | `citations.yaml` для сценария, ручная проверка выборки статей | `check`, `link`, `dump`, seed в docker compose |
| дальше | Подтверждение редакций (`activate`), Legal QA | `diff`, `fetch` (если разрешено), поддержка RAG |

---

## 11. Критерии готовности

- [ ] `python -m legal_ingestion run` с чистой базы проходит без ошибок.
- [ ] `check` — 0 ошибок, отчёт закоммичен.
- [ ] Минимальный корпус загружен хотя бы на ru и uz-Latn.
- [ ] У каждой активной редакции есть `source_url`, `retrieved_at` и `verified_by`.
- [ ] Все `citation_id` сценария находятся в базе.
- [ ] `eval/report.md` с Recall@5 по языкам; модель выбрана.
- [ ] `seed.sql.gz` поднимает базу в docker compose без ключей и интернета.
- [ ] Ни одной фиктивной статьи и ни одного выдуманного номера в корпусе и сценариях.
