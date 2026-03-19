# Survey Definition — полный путь опроса

Эту неделю я разбирал как формируется опрос — от того момента когда ты рисуешь шаги на канвасе, до того как всё это оказывается в базе данных, я прошелся как по фронту , так и по бэку

## Путь сохранения (фронт -> БД)

```
Редактор (канвас) — рисуем шаги и соединяем их
    ↓
SurveyEditorManagerService — хранит состояние канваса
— управляет шагами, соединениями, позициями
    ↓
StepPropertiesPanel — редактируем шаг
— добавляем ответы, меняем типы, пишем labels
    ↓
SurveyTransformationService.toBackendSchema()
— канвас → JSON схема
    ↓
SurveySchemaValidatorService.validate()
— валидация на фронте
    ↓
SurveyDefenitionService (Angular) → POST /surveysengine/SurveyDefinition
— Schema + VisualSchema + Title летят на бэк
    ↓
SurveyDefinitionController.AddAsync() / UpdateAsync()
    ↓
SurveyDefinitionService
— валидация Schema через JsonSchemaNetValidator
— валидация VisualSchema
— подсчёт QuestionsAmount
— base.AddAsync() → BaseEntityService
    ↓
SurveyDefinitionRepository.CreateAsync()
— генерирует SurveyId = Guid.NewGuid()
— ставит Version = 1
— INSERT в PostgreSQL
```

## Путь чтения (БД → фронт)

```
GET /surveysengine/SurveyDefinition/:id
    ↓
SurveyDefinitionRepository.GetAsync()
— SELECT из PostgreSQL
    ↓
Mapper → SurveyDefinitionResponseDTO
— Entity → DTO
    ↓
SurveyDefenitionService (Angular) → SurveyDefinitionEntity
    ↓
SurveyTransformationService.fromBackendSchema()
— JSON схема → канвас
— восстанавливает шаги, соединения, переменные
    ↓
applyVisualSchema()
— восстанавливает позиции элементов на канвасе
```

---

## Файлы которые я смотрел

### SurveyEditorManagerService (фронт)
Центральный сервис редактора, он хранит все состояние канваса через `BehaviorSubject` — все компоненты подписаны и автоматически получают обновления 

Основной функционал:
- `createNewSurvey()` — создаёт пустой опрос и сразу добавляет START шаг
- `addStep(type, x, y)` — добавляет шаг на канвас, генерирует уникальный stepId (`step_N`, `branch_N`, `end_N`)
- `removeStep(stepId)` — удаляет шаг и все его соединения
- `createConnection(source, target)` — создаёт стрелку между шагами, проверяет лимит соединений по типу шага. END не может иметь исходящих, QUESTION — максимум одно
- `updateStepPosition(stepId, x, y)` — двигает шаг и пересчитывает все соединения

Интересный момент — `syncStepCounter()`:
При загрузке существующего опроса парсит все stepId, находит максимальное число и ставит счётчик на `max + 1`, именно так новые шаги никогда не получат уже существующий ID

### StepPropertiesPanel (фронт)
Правая панель в редакторе — появляется когда кликаешь на шаг и получает шаг через `@Input()`, при изменениях триггерит событие через `@Output()`

Основной функционал:
- `addAnswer()` — добавляет новый ответ с типом INPUT и группой default
- `removeAnswer(index)` — удаляет ответ по индексу
- `updateAnswerType(index, type)` — меняет тип и автоматически добавляет нужные поля, SELECT получает `possibleVariants[]`, DATE получает `format: 'YYYY-MM-DD'`, RANGE получает `minValue`, `maxValue`, `step`
- `updateStepId()` — меняет ID шага и обновляет все соединения где этот шаг был source или target

Почти все текстовые поля мультиязычные — для каждого есть `open/close/save*Editor()`

### survey-editor.component.ts (фронт)
Главный компонент редактора, загружает опрос по ID из URL, при сохранении вызывает `toBackendSchema()`, а перед сохранением валидирует схему, если опрос уже опубликован — предлагает создать новую версию через `createNewVersion()` 

Ключевые методы:
- `loadSurvey()` — загружает опрос из бэка и восстанавливает канвас
- `saveSurvey()` — сохраняет или создаёт новую версию
- `addStepToCanvas()` — добавляет новый шаг на канвас
- `autoCreateVariablesForAnswers()` — автоматически создаёт переменные для ответов у которых нет variable

### survey-transformation.service.ts (фронт)
Самый важный сервис редактора. Преобразует между двумя форматами:
- `toBackendSchema()` — канвас → JSON для бэка. START шаг превращает в объект `start`, для каждого шага берёт `next` из реальных соединений, убирает пустые поля у ответов
- `fromBackendSchema()` — JSON с бэка → канвас. Восстанавливает шаги, строит соединения из `next` полей
- `applyVisualSchema()` — восстанавливает позиции элементов из сохранённых координат

### survey-defenition.service.ts (фронт)
Angular сервис для HTTP запросов. Наследует `BaseCRUDService` — CRUD из коробки, 
Эндпоинт: `surveysengine/SurveyDefinition`, 
Дополнительно:
- `publishSurvey()` → POST `/{id}/publish`
- `createNewVersion()` → POST `/{id}/create-version`

### SurveyDefinitionController.cs (бэк)
Контроллер с GUID. Наследует `BaseCRUDController` — CRUD из коробки,
Дополнительно:
- `POST /{id}/publish` — публикует опрос
- `POST /{id}/archive` — архивирует
- `POST /{id}/create-version` — новая версия
- `GET /{id}/versions` — все версии

### SurveyDefinitionRequestDTO.cs (бэк)
То, что летит с фронта на бэк:
- `Title` — `{ ru: "...", en: "..." }`
- `Schema` — JSON опроса (шаги, ответы, правила)
- `VisualSchema` — позиции на канвасе `{ stepId: { x, y } }`
- `CoinsAmount` — награда за прохождение

### SurveyDefinitionResponseDTO.cs (бэк)
То что бэк отдаёт на фронт. Те же поля что и в Entity плюс:
- `ActiveInstanceId` — ID активного инстанса для текущего пользователя
- `QuestionsAmount` — количество вопросов, вычисляется автоматически

### SurveyDefinitionService.cs (бэк)
Сервис с бизнес-логикой. Переопределяет `AddAsync()` и `UpdateAsync()`:
- Валидирует `Schema` и `VisualSchema`
- Считает `QuestionsAmount` — просто считает шаги с `type: "question"`
- Если статус не `Draft` — не даёт менять `Schema`
- Инвалидирует кэш после сохранения
- При публикации кэширует на 4 часа

### SurveyDefinitionEntity.cs (бэк)
Таблица в БД:
- `Schema` — jsonb
- `VisualSchema` — jsonb
- `Status` — Draft / Published / Archived
- `Version` — номер версии
- `SurveyId` — общий ID для всех версий одного опроса
- `ActiveInstanceId` — `[NotMapped]`, не хранится в БД

### SurveyDefinitionRepository.cs (бэк)
Репозиторий. Переопределяет два метода:
- `CreateAsync()` — генерирует `SurveyId = Guid.NewGuid()` и ставит `Version = 1`
- `UpdateAsync()` — синхронизирует `CoinsAmount` для всех более новых версий того же опроса

### BaseEntityService.cs (бэк)
В проекте много разных сущностей — опросы, пользователи, категории. У каждой нужен один и тот же набор операций: создать, обновить, удалить, получить. `BaseEntityService` — это шаблон который написали один раз, и все сервисы просто наследуют его вместо того чтобы писать одно и то же по 10 раз

`SurveyDefinitionService` говорит: "Беру всё из базового класса, но перед сохранением хочу ещё провалидировать схему и посчитать вопросы" — и вызывает `base.AddAsync()` когда своя логика отработала

Внутри `AddAsync()` происходит вот что:
- `_mapper.MapRequestAsync()` — DTO → Entity (превращает то что пришло с фронта в объект для БД)
- `BeforeAdding()` — хук перед сохранением, можно переопределить если нужно
- `_repo.CreateAsync()` — добавляет в EF контекст
- `_repo.SaveChangesAsync()` — INSERT в PostgreSQL
- `_mapper.MapResponseAsync()` — Entity → ResponseDTO (превращает обратно и отдаёт на фронт)
