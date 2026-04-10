# FilesMicroservice — как работает файловый обмен между микросервисами

## Как хранятся файлы?

В системе есть три типа файлов:

`File` — обычный файл, загружается целиком через HTTP

`ChunkedFile` — большой файл который загружается кусками 

`LongStorageFile` — файл в долгом хранилище, к нему обращаются другие микросервисы через RabbitMQ

---

## Контроллеры

`FileController`, `ChunkedFileController`, `LongStorageFileController` — все три наследуют `BaseCRUDController` и сами логики не содержат, просто точки входа через HTTP

```csharp
public class FileController : BaseCRUDController<int, FileRequestDTO, FileResponseDTO>
{
    // вся логика в сервисе, контроллер просто маршрутизирует
}
```

---

## Сервисы

`LongStorageFileService` — HTTP-сервис, при удалении файла физически удаляет его с диска:

```csharp
public override async Task<string> DeleteAsync(int id)
{
    var entity = await _repo.GetAsync(id);
    if (entity != null)
    {
        FilesHelper.DeleteFile(_hostingEnvironment, _fileServicesConfig, entity.Code);
    }
    return await base.DeleteAsync(id);
}
```

`LongStorageFileRPCService` — RabbitMQ-сервис, при сохранении выбирает стратегию через фабрику в зависимости от типа файла:

```csharp
switch (item.FileType)
{
    case FileType.File:
        var fileStrategy = _longStorageFileSaveFactory.GetStrategy<FileEntity>();
        result = await fileStrategy.AddAsync(item);
        break;
    case FileType.ChunkedFile:
        var chunkedFileStrategy = _longStorageFileSaveFactory.GetStrategy<ChunkedFileEntity>();
        result = await chunkedFileStrategy.AddAsync(item);
        break;
}
```

это паттерн Strategy — фабрика сама решает какую стратегию дать, сервис не знает деталей реализации

---

## Consumer

`LongStorageFileRPCConsumer` — слушает очередь RabbitMQ и передаёт запросы в `LongStorageFileRPCService`:

```csharp
public class LongStorageFileRPCConsumer 
    : RabbitMessageConsumer<int, LongStorageFileRPCRequestDTO, LongStorageFileRPCResponseDTO>
{
    // вся логика в базовом классе, просто регистрирует тип
}
```

---

## Библиотека `Microservices.Shared.FilesMicroservice.RPC`

Это готовый клиент который другие микросервисы подключают к себе чтобы работать с файлами через Rabbit — не нужно писать Rabbit логику самому

`LongStorageFileRabbitSenderService` — отправляет запросы в FilesMicroservice:

```csharp
public class LongStorageFileRabbitSenderService 
    : AbstractRabbitSenderService<int, LongStorageFileRPCRequestDTO, LongStorageFileRPCResponseDTO>
{
    // "LongStorageFileService" — имя очереди в Rabbit
}
```

`FileRequestToIdMappingModifierAction` — маппер "сохранить файл": принимает `FileCreationRequestDTO`, отправляет в Rabbit, получает `int id`:

```csharp
public async Task<int> Process(FileCreationRequestDTO source, int destination)
{
    var result = await _longStorageFileSenderService.AddAsync(new LongStorageFileRPCRequestDTO()
    {
        FileId = source.Id,
        FileType = source.FileStorageType,
    });
    return result.Id;
}
```

`IdToLongStorageFileResponseMappingModifierAction` — маппер "получить файл": принимает `int id`, возвращает полные данные о файле:

```csharp
public async Task<LongStorageFileRPCResponseDTO> Process(int source, LongStorageFileRPCResponseDTO destination)
{
    var result = await _longStorageFileSenderService.GetAsync(source);
    return result;
}
```

---

## Полная цепочка сохранения файла

```
Другой микросервис
  → FileCreationRequestDTO через Rabbit
    → LongStorageFileRPCConsumer (слушает очередь)
      → LongStorageFileRPCService.AddAsync()
        → Factory выбирает стратегию по FileType
          → стратегия сохраняет файл в хранилище
            → возвращает LongStorageFileRPCResponseDTO
              → обратно в микросервис через Rabbit
```

## Полная цепочка получения файла

```
Другой микросервис знает id файла
  → int id через Rabbit
    → LongStorageFileRPCConsumer
      → LongStorageFileRPCService.GetAsync()
        → возвращает LongStorageFileRPCResponseDTO
          → Name, Size, Code, Mime, FileUrl...
```

---

## Что я лично взял на заметку

**RPC через Rabbit** — микросервисы не ходят в FilesMicroservice по HTTP напрямую, только через очередь, это развязывает зависимости

**Паттерн Strategy + Factory** — сервис не знает как именно сохраняется файл, он только выбирает стратегию по типу — добавить новый тип файла = добавить новую стратегию, ничего не ломая

**Shared библиотека как клиент** — `.RPC` библиотека это готовый клиент, подключил через DI и работаешь с файлами не думая о Rabbit деталях

**Два маппера = два направления** — один для сохранения (Request -> Id), другой для получения (Id -> Response)
