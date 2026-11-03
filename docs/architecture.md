# Архитектура Pathlore (черновик, этап 0)

## Общая схема

```
Файлы игры (ПК владельца)
   │  распаковка в отдельную папку (UXM / Smithbox / ERDB)
   ▼
pipelines/<game>/  ── Python: извлечение предметов, текстов, координат
   │  JSON → загрузка в базу
   ▼
PostgreSQL (+ pgvector, pg_trgm)  ◄── админка: правки и проверка меток
   │
   ▼
apps/web (Next.js, TypeScript)
   ├─ карта (Leaflet, CRS.Simple, тайлы из карты мира игры)
   ├─ карточки предметов, квесты, билды, ачивки, механики, fashion
   ├─ умный поиск
   ├─ аккаунты, трекинг прогресса
   └─ оплата (Robokassa)
```

## Стек

| Часть | Выбор | Почему |
| --- | --- | --- |
| Веб-приложение | Next.js + TypeScript | Серверный рендер для SEO, одна кодовая база для ПК и телефона |
| Карта | Leaflet, CRS.Simple | Лёгкая, работает с произвольными картинками, а не только с географией |
| База | PostgreSQL + pgvector + pg_trgm | Один сервер на всё: данные, полнотекстовый, нечёткий и векторный поиск |
| Пайплайны | Python | ERDB и инструменты сообщества для Elden Ring на Python |
| ИИ | Claude Haiku через API | Только для разбора запроса и формулировки ответа из найденных строк |
| Деплой | Docker Compose на российском VPS | Персональные данные из РФ хранятся в РФ; один сервер дешевле |
| Оплата | Robokassa | Работает с самозанятыми, подписки, автоматические чеки |

## Умный поиск

Гибридный, ИИ не выдумывает ответы:

1. **Нормализация запроса.** Опечатки (pg_trgm), синонимы и сленг игроков из словаря игры.
2. **Поиск кандидатов.** Полнотекстовый + векторный (эмбеддинги описаний предметов, квестов, локаций) по выбранной игре.
3. **Ответ.** Модель получает только найденные строки и отвечает со ссылками на их id. Нет кандидатов — «не нашёл», без догадок.
4. **Обратная связь.** Кнопка «нашёл?» — главная метрика беты (цель ≥ 80%).

Бесплатно: несколько умных запросов в день; обычный поиск по названию — без ограничений.

## Схема базы (первая версия)

```
games            id, slug, name
maps             id, game_id, slug, name, layer (surface/underground/dlc), width, height, tiles_path
locales          code (ru, en, ...)

items            id, game_id, category, game_param_id, icon_path, stats jsonb
item_i18n        item_id, locale, name, description            -- из локализации игры

markers          id, map_id, kind (item/npc/boss/grace/secret/...), x, y,
                 ref_type, ref_id, source (datamine/manual/community),
                 verified bool, verified_at
quests           id, game_id, slug
quest_steps      id, quest_id, step_order, marker_id null
quest_i18n /
quest_step_i18n  ..., locale, title / text, attribution_id null

builds           id, game_id, slug, items jsonb, stats jsonb
achievements     id, game_id, steam_api_name, marker_id null
articles         id, game_id, kind (mechanic/guide), slug       -- объяснение механик
fashion_sets     id, game_id, slug, item_ids int[]
*_i18n           переводы для всего, что выше

attributions     id, source, license, url                       -- CC BY-SA и т.п.
search_docs      id, game_id, entity_type, entity_id, locale, text, embedding vector

users            id, email, created_at                          -- российский хостинг
subscriptions    id, user_id, plan, status, paid_until
user_progress    user_id, marker_id, found_at
```

## Открытые технические вопросы

- Пересчёт координат из файлов карт в координаты карты мира (этап 1, калибровка по 5–10 точкам).
- Какие файлы нужно распаковать, чтобы занять минимум места на диске E.
- Доступность из РФ выбранного хостинга и ИИ-API (этап 0, проверяет человек в России).
