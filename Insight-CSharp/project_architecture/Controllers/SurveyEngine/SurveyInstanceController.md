#  SurveyInstanceController.cs 

# Принимает Http запрос 

### Dependency Injection

```csharp
private readonly IAttemptsService _attemptsService;/

public SurveyInstanceController(
    IEntityService<Guid, CreateInstanceRequestDTO, SurveyInstanceResponseDTO> entityService,
    ISurveyInstanceService surveyInstanceService,
    ILogger<SurveyInstanceController> logger
    ) : base(entityService, logger)
{
    _surveyInstanceService = surveyInstanceService; // Dependency injection of the survey instance service
}
```

#Главное отличие от обычного контроллера — наследуется от BaseCRUDController, то есть из коробки умеет делать Create, Read, Update, Delete.
# private readonly - поле нельзя уже перезаписать после иницилизации 

## Методы

- `POST /start` — старт опроса, возвращает первый вопрос 
- `POST /{instanceId}/answer` - отправить ответ, возвращает следующий вопрос
- `GET /{instanceId}/current-step` — получить текущий вопрос 
- `POST /{instanceId}/complete` — завершить текущий вопрос
- `POST /{instanceId}/reject` — отменить опрос
- `GET /{instanceId}/history` — история всех ответов
