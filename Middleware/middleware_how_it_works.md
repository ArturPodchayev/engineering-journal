# Middleware — полный разбор
 
## Что такое Middleware
 
Каждый HTTP запрос в .NET проходит через цепочку Middleware — это как конвейер на заводе. Каждый узел что-то делает с запросом и передаёт дальше. Если узел не вызвал `await _next(context)` — запрос остановился и дальше не пошёл 

```
HTTP запрос
    ↓
ValidationErrorMiddleware
    ↓
TelegramTokenMiddleware
    ↓
JwtMiddleware
    ↓
PlatformCheckMiddleware
    ↓
Controller
    ↓
HTTP ответ
```

## ValidationErrorMiddleware — глобальный try-catch
 
```csharp
try {
    await _next(context); // запускает весь остальной конвейер
}
catch (BadAuthorizationException) → 400 Bad Request
catch (UnauthorizedException)     → 401 Unauthorized
catch (Exception)                 → пробрасывает дальше (500)
```
 
**Кто он и что он вообще делает:** это первый Middleware в цепочке
он оборачивает всё остальное в try-catch если где-то глубоко в сервисе кидается `throw new BadAuthorizationException("Failed to create user")` — оно долетает сюда и превращается в понятный JSON ответ с кодом 400
Без него каждый сервис должен был бы сам обрабатывать ошибки
 
**Пример:** в `TokenAuthorizationService` было явно прописано:
```csharp
throw new BadAuthorizationException("Failed to create user");
```
Это исключение летит вверх по стеку и доходит до `ValidationErrorMiddleware` и клиент по итогу получает:
```json
{
  "statusCode": 400,
  "message": "Failed to create user"
}
```


## TelegramTokenMiddleware — авторизация через Telegram
 
Подключается только если в конфиге включена телега:
```csharp
if (telegramConfig != null && telegramConfig.IsEnabled)
    webApplication.UseMiddleware<TelegramTokenMiddleware>();
```
 
**Что делает:** смотрит в заголовок `Authorization` если там `tma ...` — это Telegram Mini App токен. Валидирует его, достаёт userId, загружает юзера из базы и кладёт в `context.Items["User"]`
 
```
Authorization: tma eyJhbGc...
    ↓
валидация токена
    ↓
загрузка юзера из базы
    ↓
context.Items["User"] = user
    ↓
идёт дальше
```
 
Если заголовка нет — просто пишет "NO TOKEN" в лог и идёт дальше. Не блокирует. Это задача следующих Middleware
 
**Почему я в логах видел "NO TOKEN":** Telegram включён в конфиге, но я слал обычный запрос через Swagger без `tma` токена (ИИ объяснил этот момент)

## JwtMiddleware —  это базовая авторизация через обычный JWT
 
**Суть:** смотрит в заголовок `Authorization: Bearer ...`. Если токен есть — валидирует подпись, достаёт клеймы, загружает юзера из базы и кладёт в `context.Items["User"]`
 
```
Authorization: Bearer ТОКЕН
    ↓
валидация подписи токена
    ↓
достаём клеймы из токена:
    nameidentifier → userId = 11
    name           → "Artur"
    role           → "Administrator"
    emailaddress   → "test@test.com"
    OrganizationId → "1a7c524a-..."
    ↓
загружаем юзера из базы по userId
    ↓
context.Items["User"] = user
    ↓
идёт дальше
```
Кстати, важная деталь: если юзер не найден в базе по userId — создаёт призрачный объект только из клеймов. Это запасной вариант на случай если юзера удалили но токен еще валидный, полезно при тестах

## PlatformCheckMiddleware — middleware для мобильного клиента
 
**Что делает:** работает только если в запросе есть заголовок `Platform: CLIENT`, в этом случае пересчитывает `AccessDepthType` под роль `DefaultUser`
Используется когда мобильное приложение работает от имени обычного пользователя и независимо от его реальной роли в системе
 
---
 
## JwtGatewayMiddleware — проверка у каждого эндпоинта
 
Этот middleware нахлодится в Gateway (Ocelot) — не в самом NetIdentityMicroservice и он решает имеет ли право конкретный юзер вызвать конкретный эндпоинт
 
```
Запрос: GET /identity/Authorization/authorizeWithToken?token=ТОКЕН
    ↓
Парсит URL:
    service    = "identity"
    controller = "AuthorizationController"
    action     = "GET authorizeWithToken"
    ↓
Идёт в базу — ищет RoleControllerAction
для этого контроллера + action + роль юзера
    ↓
Юзер есть?
    ├── НЕТ → ищет права для роли "Unauthorized"
    │         нет прав → 401
    │         есть → устанавливает AccessDepthType → идёт дальше ✅
    └── ДА  → ищет права для роли юзера
              нет прав → 401
              есть → устанавливает AccessDepthType → идёт дальше ✅
```
 
**RoleControllerActions** — таблица в базе данных. Там записано какая роль имеет доступ к какому эндпоинту - это и есть система прав
 
**AccessDepthType** — глубина доступа к данным:
- `All` — видно все
- `Partial` — видно только свое
- `Public` — минимум

Атрибут `[RoleHasAccess("Unauthorized")]` — озночает что в системе этот эндпоинт доступен без авторизации и именно поэтому `authorizeWithToken` работает без JWT токена в заголовке

## Порядок подключения — где вообще все это настраивается
 
В `DI.cs` есть два метода:
 
**`AttachIdentityMiddlewares`** — для обычных микросервисов:
```csharp
webApplication.UseMiddleware<ValidationErrorMiddleware>();
webApplication.UseMiddleware<TelegramTokenMiddleware>(); // только если включён
webApplication.UseMiddleware<JwtMiddleware>();
webApplication.UseMiddleware<PlatformCheckMiddleware>();
```
 
**`AttachGatewayIdentityMiddlewares`** — для Gateway:
```csharp
webApplication.UseMiddleware<TelegramTokenMiddleware>();
webApplication.UseMiddleware<ValidationErrorMiddleware>();
webApplication.UseMiddleware<JwtGatewayMiddleware>();
webApplication.UseMiddleware<PlatformCheckMiddleware>();
```
 
**Почему в NetIdentityMicroservice нет JwtGatewayMiddleware:** этот сервис сам и есть авторизация — он не проверяет права, он их выдаёт
проверку прав делает Gateway для всех остальных сервисов
 
---
 
## Полный флоу нашего запроса — шаг за шагом
 
```
GET /identity/Authorization/authorizeWithToken?token=ТОКЕН
    ↓
ValidationErrorMiddleware
→ оборачивает всё в try-catch, идет дальше
 
    ↓
TelegramTokenMiddleware
→ нет заголовка "tma" → пишет "NO TOKEN" → идет дальше
 
    ↓
JwtMiddleware
→ нет заголовка "Bearer" → юзер не установлен → идет дальше
 
    ↓
PlatformCheckMiddleware
→ нет заголовка "Platform: CLIENT" → идет дальше
 
    ↓
AuthorizationController.AuthorizeWithToken()
→ TokenAuthorizationService.Process()
→ нашли токен в базе 
→ юзера нет → создали через CreateAsync
→ ActivatedOn = now
→ IsUsed = true
→ перечитали юзера из базы заново (критический фикс, без этого ломалась система)
→ CreateToken() → JWT с клеймами
→ вернули AuthorizationResponseDTO
 
    ↓
200 OK 
    {
      "email": "test@test.com",
      "token": "ТОКЕН...",
      "refreshToken": "ТОКЕН...",
      "roleId": 1,
      "refreshTimeInMinutes": 120
    }
```
 
 
## Что я в первую очередь обязан взять на заметку
 
- `context.Items["User"]` — общая шина данных запроса, middleware кладут туда юзера, контроллер достаёт, живет только один запрос
- `await _next(context)` — без этой строки запрос умирает на текущем Middleware
- Порядок подключения Middleware важен — ValidationErrorMiddleware должен быть первым чтобы ловить ошибки из всех остальных
- JWT токен можно декодировать на jwt.io — там видны все клеймы внутри
- Настройки пароля Identity в `Program.cs` — по умолчанию только минимум 6 символов, остальные требования отключены
- `TelegramTokenMiddleware` подключается условно — только если в конфиге `TelegramConfig.IsEnabled = true`

3/26/26
