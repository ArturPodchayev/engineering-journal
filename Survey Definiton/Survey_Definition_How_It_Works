# Survey Definition — как формируется опрос

Это полный путь от того как сначала рисуем опрос в редакторе до того как он оказывается в базе данных

```
Редактор (канвас) — рисуем шаги и соединяем их
    ↓
SurveyTransformationService.toBackendSchema()
— превращает всё что нарисовано в JSON схему
    ↓
SurveySchemaValidatorService.validate()
— валидация на фронте перед отправкой
    ↓
POST /SurveyDefinition (SurveyDefinitionRequestDTO)
— Schema + VisualSchema + Title летят на бэк
    ↓
SurveyDefinitionService.AddAsync() / UpdateAsync()
— валидация схемы на бэке
— подсчёт количества вопросов
— сохранение через BaseEntityService
    ↓
PostgreSQL — surveyengine."SurveyDefinitions"
— Schema (jsonb), VisualSchema (jsonb)
```

## Файлы которые я просмотрел

### survey-editor.component.ts
Основной компонент редактора, загружает опрос по ID из URL, при сохранении вызывает `transformationService.toBackendSchema()` - превращает состояние канваса в JSON. Перед сохранением валидирует схему, а после - сохраняет через `surveyService`
Ну а если опрос уже опубликован - предлагает создать новую версию

Ключевые методы:
- `loadSurvey()` — загружает опрос из бэка и восстанавливает канвас
- `saveSurvey()` — сохраняет или создаёт новую версию
- `addStepToCanvas()` — добавляет новый шаг на канвас
- `autoCreateVariablesForAnswers()` — автоматически создаёт переменные для ответов у которых нет переменной

### survey-transformation.service.ts
важный сервис в редакторе, он отвечает за преобразование между двумя форматами:
- `toBackendSchema()` — kanvas → JSON который летит на бэк
- `fromBackendSchema()` — JSON с бэка → kanvas

Что делает `toBackendSchema()`:
- Перебирает все шаги на канвасе
- START шаг превращает в объект `start` (не в `steps`)
- Для каждого шага берёт `next` из реальных соединений на канвасе, а не из stepData
- Убирает пустые поля у ответов (`variable`, `label`) чтобы не падала валидация

### SurveyDefinitionController.cs
Контроллер с GUID. Наследует `BaseCRUDController` — CRUD из коробки. Дополнительно:
- `POST /{id}/publish` — публикует опрос
- `POST /{id}/archive` — архивирует
- `POST /{id}/create-version` — создаёт новую версию
- `GET /{id}/versions` — все версии опроса

### SurveyDefinitionRequestDTO.cs
То что летит с фронта на бэк при сохранении:
- `Title` — словарь `{ ru: "...", en: "..." }`
- `Schema` — JSON опроса (шаги, ответы, правила)
- `VisualSchema` — позиции элементов на канвасе `{ stepId: { x, y } }`
- `CoinsAmount` — награда за прохождение

### SurveyDefinitionService.cs
Сервис с бизнес-логикой. Переопределяет `AddAsync()` и `UpdateAsync()` из базового класса:
- Валидирует `Schema` через `JsonSchemaNetValidator`
- Валидирует `VisualSchema`
- Считает количество вопросов через `CalculateQuestionsAmount()` — просто считает шаги с `type: "question"`
- Инвалидирует кэш после сохранения
- Если опрос не в статусе `Draft` — не даёт менять `Schema`

### SurveyDefinitionEntity.cs
Таблица в БД. Уже разбирал на первой неделе, ничего нового:
- `Schema` — jsonb, сама схема опроса
- `VisualSchema` — jsonb, позиции на канвасе
- `Status` — Draft / Published / Archived
- `Version` — номер версии, начинается с 1
- `ActiveInstanceId` — `[NotMapped]`, вычисляется из `Instances`, не хранится в БД

### BaseEntityService.cs
Универсальный базовый CRUD сервис. Внутри `AddAsync()`:
- `_mapper.MapRequestAsync()` — DTO → Entity
- `BeforeAdding()` — хук перед сохранением, можно переопределить
- `_repo.CreateAsync()` — добавляет в EF контекст
- `_repo.SaveChangesAsync()` — INSERT в PostgreSQL
- `_mapper.MapResponseAsync()` — Entity → ResponseDTO
