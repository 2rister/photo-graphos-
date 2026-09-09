# Instagram → Telegram: автопостинг и репост по ссылке

Документ-исследование и план реализации. Дата: 2026-09-09.

## 0. Выбранная конфигурация (подтверждена владельцем)

- Instagram-аккаунт — **Professional** → идём **только официальным Graph API**.
- Репостим **только свои посты** → скрапер (instagrapi/instaloader) **не нужен**.
- Публикация **сразу, без подтверждения** → предпросмотр в ЛС не делаем
  (остаётся как опциональная доработка, п. 7).

Разделы 2.2 (неофициальные библиотеки) и 2.3 (no-code) оставлены как
обоснование выбора и запасной вариант — в реализацию не входят.

## 1. Ключевые факты (проверено по документации)

### 1.1. Вебхука на «новый пост» у Instagram НЕТ
Поля вебхуков Instagram Platform: `comments`, `live_comments`, `mentions`,
`message_echoes`, `message_reactions`, `messages`, `messaging_handover`,
`messaging_optins`, `messaging_policy_enforcement`, `messaging_postbacks`,
`messaging_referral`, `messaging_seen`, `response_feedback`, `standby`,
`story_insights`.

Событие «владелец опубликовал новый пост» отсутствует.
→ **Единственный официальный способ узнать о новом посте — периодический
опрос (polling) `GET /me/media`.**

### 1.2. Официальный API
- Нужен **Professional-аккаунт** (Business или Creator). Личные аккаунты
  официального доступа не имеют (Basic Display API закрыт).
- Scope: `instagram_business_basic` (заменил `business_basic`, старые значения
  задепрекейчены 27.01.2025).
- **App Review НЕ нужен**, если приложение остаётся в Development mode и
  работает только с собственным аккаунтом (аккаунт добавлен как
  admin/developer/tester приложения). Токен генерируется прямо в дашборде:
  Instagram → API setup with Instagram login → Generate access token.
- `GET /me/media`: до 10 000 последних медиа, поддерживает time-based
  пагинацию (`since` / `until`).
- Rate limit: **200 вызовов на пользователя в час**. Опрос раз в 5 минут =
  12 вызовов/час — с огромным запасом.
- `media_url` — **временный подписанный CDN-URL**, истекает через часы/дни.
  → Медиа нужно скачивать и заливать в Telegram байтами, а не отдавать ссылкой.
- Stories через `/me/media` не приходят — отдельный эндпоинт `/me/stories`.

### 1.3. Telegram Bot API
- Загрузка ботом: **до 50 МБ**, скачивание — до 20 МБ.
  Свой Local Bot API server (`tdlib/telegram-bot-api`) снимает лимит до 2 ГБ.
- `sendMediaGroup`: альбом **2–10 элементов**. Карусель IG бывает до 20 →
  резать на 2 альбома.
- Подпись к медиа: **1024 символа**. Обычное сообщение: 4096.
  Подпись IG бывает до 2200 → длинную резать или слать отдельным сообщением.

## 2. Инструменты: что реально есть на GitHub

### 2.1. Готового решения «мои новые посты → мой канал» нет
Ближайшие по смыслу репозитории (полезны как референс, не как готовый продукт):

| Репозиторий | ★ | Что делает | Зачем нам |
|---|---|---|---|
| `subinps/Instagram-Bot` | 536 | TG-бот, качает почти всё из IG по ссылке | референс для сценария «прислал ссылку» |
| `2pai/instastory-monitor-telegram` | 57 | мониторит сторис аккаунта → шлёт в TG | референс polling-цикла |
| `zerox9dev/Vidzilla` | 69 | TG-бот-загрузчик (aiogram + yt-dlp) | референс архитектуры aiogram |
| `gth-ai/reclip-telegram-bot` | 53 | то же на yt-dlp | референс |
| `Wikidepia/InstaFix` | 1029 | фикс IG-эмбедов для TG/Discord (АРХИВИРОВАН) | вариант «просто ссылка с превью» |
| `seirenkr/OGInstagram` | 72 | живой аналог InstaFix (Go) | вариант «просто ссылка с превью» |
| `sokomishalov/skraper` | 348 | скрапер многих соцсетей (Kotlin) | не наш стек |

### 2.2. Библиотеки доступа к Instagram

**Официальные (рекомендуется):** обычный HTTP к Graph API, SDK не нужен —
хватит `httpx`.

**Неофициальные (fallback, нарушают ToS Instagram, риск бана аккаунта):**
- `subzeroid/instagrapi` — самая живая private-API библиотека (Python 3.10+),
  есть async-версия `subzeroid/aiograpi`.
- `instaloader/instaloader` — 13.1k★, последний коммит 26.07.2026, качает
  публичные профили.
- `mikf/gallery-dl` — качает профили/посты/reels/stories, но с 2023 требует
  cookies залогиненного браузера.
- `yt-dlp` — для видео/reels.
- `DIYgod/RSSHub` — Instagram-роуты **сейчас сломаны** (issues #19864, #17826),
  как источник ненадёжен.

Общая проблема всех скраперов: Instagram ротирует внутренний GraphQL `doc_id`,
поэтому они ломаются примерно раз в 2–4 недели.

### 2.3. No-code альтернативы (если не хочется своего кода)
- **Make.com** — есть готовый шаблон «Send new Instagram media (images and
  videos) to Telegram Bot».
- **Zapier** — триггер «New Media Posted in my Account» (Instagram for
  Business) + действие Telegram.
- **n8n** — встроенный Facebook Trigger для Instagram события «новый пост» не
  даёт (только комменты/DM/mentions); community-нода
  `MookieLian/n8n-nodes-instagram` + Schedule Trigger с опросом `/me/media`.

Минусы no-code: карусели и длинные подписи обрабатываются криво, сценарий
«прислал ссылку → магия» там не собирается нормально, и платно на объёме.

## 3. Рекомендуемая архитектура

Один Python-сервис, два входа, общий конвейер публикации.

```
                    ┌──────────────────────────┐
   каждые 5 мин ───▶│ poller: GET /me/media    │──┐
                    │ (since = last_ts)        │  │
                    └──────────────────────────┘  │
                                                  ▼
   ссылка в ЛС боту ─▶ resolve по permalink ─▶ ┌─────────────────┐
                       среди своих медиа       │ dedup (SQLite)  │
                                               └────────┬────────┘
                                                        ▼
                                          ┌──────────────────────────┐
                                          │ download media (temp CDN)│
                                          │ build caption            │
                                          │ send to @channel         │
                                          └──────────────────────────┘
```

**Про «прислал ссылку → магия»:** если пост свой, скрапер не нужен вообще.
`/me/media` возвращает поле `permalink` — по shortcode из присланной ссылки
находим медиа в собственном списке и публикуем тем же кодом, что и автопост.
Скрапер (instagrapi) понадобится, только если надо репостить **чужие** посты.

### 3.1. Стек (что ставить)

```
python 3.12
aiogram>=3.15          # Telegram-бот (async, актуальный)
httpx                  # Graph API + скачивание медиа
apscheduler            # планировщик опроса
aiosqlite              # дедупликация опубликованного
pydantic-settings      # конфиг из .env
tenacity               # ретраи на 429/5xx
```
Опционально:
```
ffmpeg (системный)     # нормализация видео под TG
tdlib/telegram-bot-api # локальный Bot API server, если видео > 50 МБ
```
`instagrapi` / `aiograpi` в выбранной конфигурации **не устанавливаем**.

### 3.2. Схема данных (SQLite)

```sql
CREATE TABLE posted (
  ig_media_id   TEXT PRIMARY KEY,
  shortcode     TEXT UNIQUE,
  permalink     TEXT,
  tg_message_id INTEGER,
  media_type    TEXT,
  posted_at     TEXT,
  created_at    TEXT
);
CREATE TABLE state (key TEXT PRIMARY KEY, value TEXT);  -- last_seen_timestamp, token, token_expires_at
```

## 4. План реализации по этапам

### Этап 0. Подготовка доступов (ручной, без кода)
1. Перевести Instagram в Professional (Business/Creator), если ещё не.
2. Создать Meta App, добавить продукт Instagram, оставить в Development mode.
3. Сгенерировать long-lived access token (60 дней) в дашборде.
4. Создать Telegram-бота через @BotFather, добавить админом в канал.
5. Собрать `.env`: `IG_ACCESS_TOKEN`, `IG_USER_ID`, `TG_BOT_TOKEN`,
   `TG_CHANNEL_ID`, `TG_OWNER_ID`, `POLL_INTERVAL_SEC`.

### Этап 1. Скелет
- Структура пакета, конфиг на pydantic-settings, логирование, Dockerfile,
  docker-compose (сервис + volume под SQLite).

### Этап 2. Instagram-клиент
- `GET /{ig_user_id}/media?fields=id,caption,media_type,media_product_type,
  media_url,thumbnail_url,permalink,timestamp,children{id,media_type,media_url}`
- пагинация по `since`/`until`, ретраи, обработка 429.
- refresh long-lived токена (`/refresh_access_token`) по расписанию раз в
  ~50 дней + предупреждение владельцу в ЛС, если протух.

### Этап 3. Рендер и публикация в Telegram
- IMAGE → `sendPhoto`; VIDEO/REELS → `sendVideo`; CAROUSEL_ALBUM →
  `sendMediaGroup` чанками по 10.
- Подпись: caption из IG + ссылка на оригинальный пост; если > 1024 —
  укороченная подпись + полный текст отдельным сообщением.
- Скачивание медиа во временный файл (CDN-URL истекает), проверка размера
  против лимита 50 МБ, понятная ошибка владельцу если больше.

### Этап 4. Poller
- APScheduler каждые N минут (по умолчанию 5).
- Берём медиа с `timestamp > last_seen`, фильтруем по таблице `posted`,
  публикуем от старых к новым, обновляем `last_seen`.
- Первый запуск: не заливать весь архив — ставим `last_seen = now`
  (или явный флаг `--backfill N`).

### Этап 5. Репост по ссылке
- Хэндлер в боте: принимает ссылки вида
  `instagram.com/p/<code>`, `/reel/<code>`, `/tv/<code>`, `share/`-редиректы.
- Парсим shortcode → ищем среди `/me/media` по `permalink` → тот же конвейер.
- Если уже публиковалось — отвечаем ссылкой на существующее сообщение в канале
  и спрашиваем «опубликовать ещё раз?».
- Доступ к боту — только для `TG_OWNER_ID` (whitelist).

### Этап 6. Эксплуатация
- Health-эндпоинт/heartbeat, алерты владельцу в ЛС при ошибках,
  graceful-обработка 429 Telegram (flood control), деплой на VPS в Docker.

### Этап 7 (опционально, за рамками текущего объёма)
- Локальный Bot API server, если понадобятся видео > 50 МБ.
- Режим предпросмотра: бот сначала шлёт пост владельцу с кнопками
  «Опубликовать / Пропустить» вместо мгновенной публикации.
- Фильтры по хэштегам, кнопка «Открыть в Instagram» под постом.
- Stories (`/me/stories`) — отдельным потоком, если понадобится.
- Fallback на `instagrapi` для чужих постов — только если изменится задача.

## 5. Риски
| Риск | Митигация |
|---|---|
| ~~Аккаунт не Professional~~ | не актуально: аккаунт уже Professional |
| Токен протухает (60 дней) | авто-refresh + алерт владельцу |
| `media_url` истёк | скачивать сразу, не хранить ссылки |
| Видео > 50 МБ | local Bot API server или ссылка вместо файла |
| Опрос раз в 5 мин | задержка публикации до 5 мин — не realtime by design |
| Скрапинг | в выбранной конфигурации не используется — риска нет |
