# Week 1 — 3–7 марта 2026

**One sentence that captures this week:** Первая неделя — поднял с нуля весь production-like стек из 9 сервисов, разобрал полную цепочку Survey Engine от HTTP запроса до JSON в базе данных и задокументировал архитектуру.

---

## What I built / worked on

**День 1 — Окружение и онбординг:**
- Клонировал репозитории `Microservices` и `insight`, разобрался со структурой монорепозитория
- Поднял 9 Docker контейнеров: OcelotApiGateway, NetIdentityMicroservice, FilesMicroservice, SurveyEngineMicroservice, TelegramBotMicroservice + PostgreSQL, RabbitMQ, NATS, pgAdmin
- Разобрался с точками входа: API Gateway на `localhost:8080`, pgAdmin на `localhost:5050`, RabbitMQ на `localhost:15672`, NATS на `localhost:8222`

**Дни 2-5 — Survey Engine:**
- Разобрал полную архитектуру Survey Engine — прошёл цепочку `Controller → Interface → Service → Engine → Entity → JSON`
- Прокомментировал код `SurveyInstanceController.cs` — каждая строка с объяснением
- Нашёл Survey JSON в базе данных через pgAdmin — таблица в схеме `surveyengine`
- Написал 7 markdown документов на GitHub в `project_architecture/Controllers/SurveyEngine/`
- Запустил фронт `MicroservicesAdmin` на `localhost:4200` — Angular + nx workspace

---

## One number

**Метрика:** 9 контейнеров поднято + 6 файлов задокументировано за первую неделю  
**Контекст:** Полная цепочка Survey Engine от контроллера до JSON схемы в базе данных разобрана и задокументирована

---

## Key concepts I learned this week

- **Dependency Injection** — контроллер не создаёт сервисы сам, получает их через конструктор. Позволяет легко подменять реализацию в тестах
- **Interface vs Implementation** — `ISurveyInstanceService` это контракт, `SurveyInstanceService` это реализация. Контроллер работает через интерфейс
- **Idempotency** — защита от двойной отправки. Если пользователь нажал кнопку дважды — второй запрос игнорируется по ключу `instanceId + stepId + хэш ответа`
- **Event-driven architecture** — после завершения опроса публикуется `SurveyCompletedEvent` в очередь. Обработчики (`WalletHandler`, `OnboardingHandler`, `SegmentHandler`) реагируют независимо
- **JSONLogic** — движок правил который читает условия из JSON и определяет следующий шаг опроса
- **jsonb в PostgreSQL** — JSON прямо в базе данных, можно делать запросы внутрь JSON структуры
- **Pointer pattern** — `SurveyInstance.Pointer` хранит ID текущего шага как закладка в книге. Обновляется после каждого ответа
- **Caching** — схема опроса кэшируется на 4 часа через `IMemoryCache` чтобы не ходить в базу на каждый запрос

---

## Questions I asked the team (and what I learned)

- Q: Как запустить проект локально? → A: Получил инструкцию с docker compose командой и скриптами pg_backup
- Q: Билд завис, что делать? → A: Разраб подсказал запускать через Git Bash если не работает обычный терминал
- Q: Какой следующий шаг после изучения контроллеров? → A: Разобрать SurveyInstanceService и Survey JSON структуру, протестировать админ панель

**Energy level this week: 8/10**  
Первая неделя — overwhelming но продуктивная. Архитектура начала складываться в единую картину к концу недели 
