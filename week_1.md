# Week 1 — 3–7 марта 2026

**One sentence that captures this week:**
Первый день — не писал код, но поднял с нуля весь production-like стек из 8 сервисов и разобрался в архитектуре проекта.

---

## What I built / worked on

Первый день: фокус на онбординге и развёртывании окружения.

- **Explored codebase structure:** Монорепозиторий `Microservices` (C# / .NET 8) + отдельный репо `insight` с микросервисами для платформы опросов. Репо `insight` нужно клонировать внутрь `Microservices/Microservices/insight/` — это не очевидно из документации.
- **Services I identified:** OcelotApiGateway, NetIdentityMicroservice, FilesMicroservice, SurveyEngineMicroservice, TelegramBotMicroservice + инфраструктура: PostgreSQL, RabbitMQ, NATS, pgAdmin
- **Integration points I found:** Все сервисы общаются через RabbitMQ и NATS. Единая точка входа — Ocelot API Gateway на порту 8080. Одна общая база данных `mainbase` в PostgreSQL.
- **Managed to run locally:** ✅ Yes — что заблокировало: Docker зависал при параллельном билде всех 5 сервисов одновременно, решилось билдом по одному сервису.

---

## What was non-obvious

- Репозиторий `insight` нужно клонировать **внутрь** папки `Microservices/Microservices/` — это нигде явно не написано, выяснилось только когда docker-compose упал с ошибкой `file not found`
- Docker на Windows плохо справляется с параллельным билдом .NET проектов — RAM и CPU показывали 0%, хотя билд якобы шёл. Решение: собирать по одному сервису
- Скрипт `pg_backup.sh` не работает с путями в формате Git Bash (`/c/Users/...`) — нужно передавать Windows-пути (`C:\\...`) или класть файл рядом со скриптом
- `filesmicroservice` висит в статусе `unhealthy` — это зависимость от `netidentitymicroservice`, норма при старте

---

## One number

**Метрика:** 8 контейнеров поднято с нуля за ~3 часа
**Контекст:** Включая отладку ошибок с путями, перезапуски Docker и поиск правильного расположения репозиториев

---

## Technical decision of the week

**Decision:** Билдить сервисы по одному вместо `up --build` всех сразу
**Why:** Docker Desktop на Windows зависал при параллельной компиляции 5 .NET проектов — процесс уходил в 0% CPU/RAM и не двигался
**What I'd do differently:** Сразу спросить у команды есть ли известные проблемы с локальным билдом на Windows

---

## AI usage this week

- **Used AI as a crutch:** Пошагово следовал инструкциям при развёртывании, не всегда понимая что происходит
- **Used AI correctly:** Диагностика ошибок (`file not found`, `EOF`, неправильные пути) — AI объяснял причину и предлагал решение, что помогло разобраться в структуре проекта

---

## Questions I asked the team (and what I learned)

1. **Q:** Как запустить проект локально? → **A:** Получил инструкцию с docker compose командой и скриптами pg_backup
2. **Q:** Билд замкнул, что делать? → **A:** Разраб подсказал запускать через Git Bash если не работает обычный терминал

---

## Questions I WANTED to ask but didn't (and why)

- Почему `insight` нужно клонировать внутрь `Microservices` — не спросил, потому что выяснил сам через ошибку
- Что значит `filesmicroservice unhealthy` — не спросил, не хотел задавать слишком много вопросов в первый день
- Какой сервис является "точкой входа" для разработки — пока не понял с чего начинать читать код

---

## What I'll do differently next week

- Сначала читать все `.md` файлы в корне репо перед тем как что-то запускать
- Задавать вопросы смелее — лучше спросить сразу чем тратить час на отладку
- Начать изучать структуру `SurveyEngineMicroservice` — это главный сервис проекта
- Открыть `Microservices.sln` в Visual Studio и пройтись по архитектуре

---

**Energy level this week: 7/10**
Онбординг всегда немного overwhelming, но приятно что окружение поднялось и всё работает.
