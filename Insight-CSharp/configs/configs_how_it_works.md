# Конфиги в .NET микросервисах

## Как хранятся конфиги?

В .NET конфиги хранятся не в `.env` файлах (это больше Node.js история), а в `appsettings.json`

Когда Docker запускает контейнер с переменной `ASPNETCORE_ENVIRONMENT=insight` — он читает оба файла и соединяет их. Значения из `appsettings.insight.json` перезаписывают значения из дефолтного `appsettings.json`

Разница между двумя файлами в нашем проекте — лишь во времени жизни токена, использовании TelegramConfig и базовой роли при регистрации. Всё остальное одинаковое

Это позволяет иметь одну кодовую базу но разное поведение для разных окружений — dev, insight, academy и aura 

---

## IBaseConfig и IAuthConfig — зачем они нужны?

оба пустые, там вообще нет никаких методов:

```csharp
public interface IBaseConfig { }   // для обычных микросервисов
public interface IAuthConfig { }   // только для NetIdentityMicroservice
```

Это marker interface — пустой интерфейс, который говорит системе: "этот класс — конфиг, зарегистрируй его автоматически"

Без него пришлось бы в `Program.cs` прописывать каждый конфиг вручную:
```csharp
services.Configure<JwtConfig>(configuration.GetSection("JwtConfig"));
services.Configure<TelegramConfig>(configuration.GetSection("TelegramConfig"));
services.Configure<RabbitConfig>(configuration.GetSection("RabbitConfig"));
// ... и так для каждого нового конфига
```

с marker interface — `AddConfigs()` делает все это сам, буквально 

---

## Как работает AddConfigs?

Когда в `Program.cs` вызывается `builder.Services.AddConfigs(builder.Configuration)` — он ищет все классы которые реализуют `IBaseConfig` и для каждого регистрирует конфиг автоматически

очень важно: имя класса должно совпадать с секцией в `appsettings.json`:

```
class JwtConfig      → читает секцию "JwtConfig"
class TelegramConfig → читает секцию "TelegramConfig"
class RabbitConfig   → читает секцию "RabbitConfig"
```

если создать новый класс `MyNewConfig : IBaseConfig` — он автоматически подхватится. В `Program.cs` ничего добавлять не нужно

---

## Как выглядит сам конфиг-класс

```csharp
// TelegramConfig.cs
public class TelegramConfig : IBaseConfig  // ← marker interface
{
    public bool IsEnabled { get; set; } = false;        // ← дефолтное значение
    public string BotToken { get; set; } = string.Empty;
}
```

```csharp
// JwtConfig.cs
public class JwtConfig : IAuthConfig  // ← для NetIdentityMicroservice
{
    public string Secret { get; set; } = string.Empty;
    public string Issuer { get; set; } = string.Empty;
    public string Audience { get; set; } = string.Empty;
    public string Subject { get; set; } = string.Empty;
    public int ExpirationTime { get; set; } = 30;  // дефолт 30 минут
}
```

Дефолтные значения важны — если секция не найдена в appsettings, свойство получит дефолт а не null

---

## Внедрение конфига в сервис через IOptions

После регистрации конфиг внедряется в любой сервис через `IOptions<T>`:

```csharp
public class TokenAuthorizationService
{
    private readonly JwtConfig _jwtConfig;

    public TokenAuthorizationService(
        IOptions<JwtConfig> jwtConfig,  // ← DI контейнер сам найдёт и подставит
        ...
    )
    {
        _jwtConfig = jwtConfig.Value;  // .Value достаёт реальный объект из обёртки
    }

    public async Task Process(...)
    {
        managedUser.RefreshTokenExpiryTime = DateTime.UtcNow
            .AddMinutes(_jwtConfig.ExpirationTime);  // ← 44640 минут для insight окружения
    }
}
```


## Внедрение конфига в Middleware

В Middleware конфиг читается немного иначе — напрямую из `WebApplication`:

```csharp
public static WebApplication AttachIdentityMiddlewares(this WebApplication webApplication)
{
    var telegramConfig = webApplication.Configuration
        .GetSection("TelegramConfig")
        .Get<TelegramConfig>();

    if (telegramConfig != null && telegramConfig.IsEnabled)
    {
        webApplication.UseMiddleware<TelegramTokenMiddleware>();
    }
    ...
}
```

Именно поэтому `TelegramTokenMiddleware` подключается только в некоторых окружениях — в `appsettings.insight.json` стоит `IsEnabled: true`, а в базовом `appsettings.json` — `false`

---

## Полная цепочка от файла до кода

```
appsettings.insight.json
  "TelegramConfig": { "IsEnabled": true, "BotToken": "..." }
        ↓
ASPNETCORE_ENVIRONMENT=insight → читается appsettings.insight.json
        ↓
AddConfigs() находит класс TelegramConfig : IBaseConfig
        ↓
services.Configure<TelegramConfig>(configuration.GetSection("TelegramConfig"))
        ↓
В DI.cs читаем конфиг и подключаем Middleware условно:
  if (telegramConfig.IsEnabled) → UseMiddleware<TelegramTokenMiddleware>()
        ↓
TelegramTokenMiddleware подключается к pipeline 
```

---

## Что я лично взял на заметку:

- **Marker interface** — пустой интерфейс как метка для автоматической регистрации
- **Имя класса = имя секции** — `JwtConfig` читает секцию `"JwtConfig"` из appsettings
- **Слои конфигов** — `appsettings.json` база, `appsettings.{env}.json` перезаписывает
- **`IOptions<T>`** — стандартный способ внедрить конфиг в сервис через DI
- **`.Value`** — нужно вызвать чтобы достать реальный объект из `IOptions<T>` обёртки
- **Дефолтные значения в классе** — страховка если секция не найдена в appsettings
