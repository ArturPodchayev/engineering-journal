# Week 4 — 23–27 марта 2026

**One sentence that captures this week:** Четвёртая неделя — разобрал систему временных токенов, пофиксил баг с повторным входом, изучил всю цепочку Middleware и написал свой первый Middleware с нуля

---

## What I built / worked on

**Этап 1 — Temporary Token Flow:**
- Переключился на ветку `feature/temporary-user` разраба
- Пересобрал `netidentitymicroservice` в Docker
- Нашёл и прогнал оба метода в Swagger — `GenerateToken` (200 OK) и `AuthorizeWithToken` (разбирали баги)
- Разобрал полный флоу: как токен генерируется, как хранится в базе, как авторизует пользователя
- Написал markdown документ на GitHub

**Этап 2 — Баг с повторным входом:**
- Разраб нашёл баг: токен сгорал после первого использования, хотя должен работать пока не истечёт `ExpiryInMinutes`
- Нашёл проблему в коде — строка `tempToken.IsUsed = true` стояла вне if/else блока и выполнялась всегда
- Фикс: заменил `tempToken.IsUsed = true` на `tempToken.ActivatedOn ??= DateTime.UtcNow` — токен больше не сгорает после каждого входа
- Протестировал: 200, 200, 200 — после истечения времени 400 
- Создал ветку `feature/temporary-user-artur`, запушил, открыл PR в ветку разраба

**Этап 3 — Изучение Middleware:**
- Разобрал все Middleware в проекте: `ValidationErrorMiddleware`, `TelegramTokenMiddleware`, `JwtMiddleware`, `PlatformCheckMiddleware`, `JwtGatewayMiddleware`
- Понял порядок подключения через `DI.cs` — `AttachIdentityMiddlewares` и `AttachGatewayIdentityMiddlewares`
- Декодировал JWT токен на jwt.io — увидел все клеймы внутри
- Написал markdown документ на GitHub

**Этап 4 — Написал свой первый Middleware:**
- Задача: реализовать `TemporaryTokenAuthMiddleware` — перехватывает запрос, если роль Unauthorized и в URL есть токен — авторизует через `authorizeWithToken` и пускает дальше
- Написал Middleware с нуля: проверка условия → HTTP запрос к NetIdentity → парсинг JWT → обновление контекста → вшивание JWT в Response Headers
- Подключил в `DI.cs` для Gateway и для обычных микросервисов
- В процессе отладки поймал и решил несколько проблем:
  - Бесконечный цикл — Middleware вызывал сам себя через `authorizeWithToken`, добавил исключение по пути
  - Символ `+` в base64 токене превращался в пробел при передаче через query string — добавил `.Replace(" ", "+")`
  - `Headers are read-only` — нельзя добавлять заголовки после того как ответ начал отправляться, переделал через `context.Response.OnStarting()`

---

## Technical decision of the week

**Decision:** Перед пушем проверять через `git diff` что в коммит попали только нужные файлы

**Why:** Случайно закоммитил вложенный git репозиторий (`Microservices/insight`) и лишний файл с длинным именем — пришлось чистить через `git rm --cached`

**What I'd do differently:** Всегда делать `git diff --name-only` перед `git add .`

---

## Key concepts I learned this week

- **Middleware pipeline** — каждый HTTP запрос проходит цепочку узлов. Каждый узел либо передаёт дальше через `await _next(context)`, либо останавливает запрос
- **context.Items["User"]** — общая шина данных запроса. Middleware кладут туда юзера, контроллер достаёт. Живёт только один запрос
- **JWT клеймы** — данные зашитые внутрь токена при генерации. Декодируются на jwt.io. Именно из них JwtMiddleware восстанавливает юзера при каждом запросе
- **IHttpClientFactory** — правильный способ делать HTTP запросы из Middleware. Управляет пулом соединений, не нужно создавать `HttpClient` руками
- **context.Response.OnStarting()** — регистрирует колбэк который выполняется перед началом отправки ответа. Единственный способ добавить заголовки после `await _next()`
- **`??=` оператор** — присваивает значение только если переменная null. `tempToken.ActivatedOn ??= DateTime.UtcNow` — устанавливает время только при первом входе
- **base64 в query string** — символ `+` в URL означает пробел. Токены в base64 содержат `+`, поэтому при чтении из `Query["token"]` нужно делать `.Replace(" ", "+")`

---

## AI usage this week

- **Used AI correctly:** Отладка Middleware — разбирали логи вместе, находили причины ошибок по цепочке
- **Used AI correctly:** Объяснение концептов — Middleware pipeline, клеймы, IHttpClientFactory
- **Used AI as a crutch:** Скелет Middleware написал AI — в следующий раз попробую написать структуру сам и только потом сверить

---

## Questions I asked the team (and what I learned)

- Q: На каком микросервисе тестировать Middleware? → A: "Бэк отлавливает путь и проверяет наличие токена на любом запросе"
- Q: Где искать GenerateToken и AuthorizeWithToken? → A: Ветка `feature/temporary-user`, репозиторий Microservices

---

## Wins this week 🏆

- Второй PR в жизни — баг с токеном пофиксил самостоятельно
- Написал Middleware с нуля — реальный production код в shared библиотеке
- Разобрал как работает вся авторизационная система проекта от и до

---

**Energy level this week: 9/10** — насыщенная неделя, много нового, Middleware почти доделан — доделаю в понедельник
