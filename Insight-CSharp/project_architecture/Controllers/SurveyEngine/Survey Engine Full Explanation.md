# Survey Engine — Полный разбор архитектуры

## Общая цепочка

```
Пользователь нажимает "начать опрос"
            ↓
SurveyInstanceController   — принимает HTTP запрос, передаёт дальше
            ↓
ISurveyInstanceService     — контракт (что умеет сервис)
            ↓
SurveyInstanceService      — вся бизнес-логика
            ↓
JsonLogicEngine            — решает на какой вопрос идти дальше
            ↓
SurveyDefinitionEntity     — модель данных, хранит Schema в базе
            ↓
Survey JSON                — сама схема опроса в PostgreSQL (jsonb)
```

---

## 1. SurveyInstanceController

Контроллер — это мост. Принимает HTTP запрос и передаёт в сервис. Сам ничего не решает

Главное отличие от обычного контроллера — наследуется от `BaseCRUDController`, то есть из коробки умеет делать Create, Read, Update, Delete

**Endpoints:**
- `POST /start` — начать опрос, возвращает первый вопрос
- `POST /{instanceId}/answer` — отправить ответ, возвращает следующий вопрос
- `GET /{instanceId}/current-step` — получить текущий вопрос
- `POST /{instanceId}/complete` — завершить опрос
- `POST /{instanceId}/reject` — отменить опрос
- `GET /{instanceId}/history` — история всех ответов

---

## 2. ISurveyInstanceService

Интерфейс — это контракт. Говорит "вот что умеет делать сервис" но не говорит КАК. Контроллер работает именно с интерфейсом, а не с конкретным классом — это позволяет легко подменять реализацию в тестах

Сам интерфейс наследует `IEntityService` — значит сервис умеет и базовый CRUD и всё что перечислено ниже

**Методы:**
- `StartSurvey` — начать опрос
- `SubmitAnswerAsync` — отправить ответ на текущий вопрос
- `GetCurrentStepAsync` — получить текущий вопрос
- `CompleteAsync` — завершить опрос
- `RejectAsync` — отменить опрос
- `GetHistoryAsync` — история всех ответов и событий
- `ProcessTimeoutAsync` — обработка таймаута, вызывается фоновой задачей

---

## 3. SurveyInstanceService

`SurveyInstanceService` — это сервис который управляет жизненным циклом прохождения опроса.

Когда пользователь стартует опрос — сервис проверяет авторизацию, проверяет доступность опроса для конкретного пользователя, создаёт `SurveyInstance` в базе и устанавливает `Pointer` на первый шаг из схемы

Когда пользователь отвечает — сервис валидирует что ответ пришёл именно на текущий шаг (через `Pointer`), защищается от дублей через `idempotencyKey`, сохраняет ответ в `Result`, обновляет переменные в `Vars` для ветвления, и передаёт всё это в `JsonLogicEngine` который возвращает ID следующего шага. `Pointer` обновляется на этот шаг

Если следующий шаг финальный — статус инстанса меняется на `Completed` и публикуется `SurveyCompletedEvent` в очередь. Обработчики этого события занимаются начислением монет, онбордингом и обновлением сегментов

Схема опроса кэшируется на 4 часа через `IMemoryCache` чтобы не делать запрос в базу на каждый ответ пользователя

### Ключевые куски кода

```csharp
// idempotencyKey — уникальный ключ из instanceId + stepId + хэша ответа
// если пользователь отправил одинаковый ответ дважды — второй игнорируется
var idempotencyKey = $"answer:{instanceId}:{request.StepId}:{request.Value.RootElement.GetRawText().GetHashCode()}";
```

```csharp
// Pointer — указатель на текущий шаг опроса (как закладка в книге)
// если ответ пришёл не на текущий шаг — выбрасываем ошибку
if (instance.Pointer != request.StepId)
    throw new InvalidOperationException(...);
```

```csharp
// спрашиваем JsonLogicEngine — куда идти дальше?
var nextStep = await _ruleEngine.GetNextStepAsync(schemaDoc, request.StepId, combinedData);
instance.Pointer = nextStep;
```

### Статусы инстанса

| Метод | Кто вызывает | Статус |
|-------|-------------|--------|
| `CompleteAsync` | пользователь сам завершил | `Completed` |
| `RejectAsync` | пользователь отменил | `Rejected` |
| `ProcessTimeoutAsync` | фоновая задача (30 мин) | `Expired` |

### После завершения опроса

```
SurveyCompletedEvent → очередь
        ↓
OnboardingSurveyCompletedHandler  — если онбординг
WalletSurveyCompletedHandler      — начисление монет
SegmentSurveyCompletedHandler     — обновление квоты сегмента
```

---

## 4. JsonLogicEngine
`JsonLogicEngine` — это мозг движка опросов. Его единственная задача — решить куда идти после того как пользователь ответил на вопрос. Он читает правила из Survey JSON и вычисляет следующий шаг
Получает схему опроса, текущий шаг и переменные пользователя — и возвращает ID следующего шага. Сам не знает ничего про опрос, просто читает правила и вычисляет результат
### Два случая поля "next"

**Простой — следующий шаг фиксирован:**
```json
"next": "feedback"
```

**Сложный — ветвление по условию:**
```json
"next": {
  "if": [
    [{ "==": [{ "var": "score" }, "1"] }, "low_step"],
    [{ ">=": [{ "var": "score" }, "4"] }, "high_step"]
  ],
  "else": "feedback"
}
```
Правила перебираются сверху вниз — возвращается первый шаг у которого условие `true`. Если ни одно не сработало — идём в `else`

### Выбор стратегии по сложности

```csharp
return parsed.Complexity switch
{
    < 3  => EvaluateSimple(...),      // простое — быстрый прямой метод
    < 10 => EvaluateCompiled(...),    // среднее
    _    => EvaluateInterpreted(...)  // сложное — интерпретация с кэшем
};
```

---

## 5. SurveyDefinitionEntity

Модель данных которая описывает как опрос хранится в базе. Каждое поле = колонка в таблице `surveyengine."SurveyDefinitions"`

### Ключевые поля

| Поле | Тип | Описание |
|------|-----|----------|
| `Schema` | jsonb | Сама схема опроса — Survey JSON |
| `VisualSchema` | jsonb | Расположение шагов в редакторе |
| `QuestionsAmount` | int | Кол-во вопросов, считается автоматически |
| `CoinsAmount` | int | Награда за прохождение |
| `Status` | string | Draft / Published / Archived |
| `Version` | int | Версия, неизменна после публикации |

### Жизненный цикл опроса

- `Draft` — черновик, пользователи не видят
- `Published` — опубликован, можно проходить
- `Archived` — больше не активен

---

## 6. Survey JSON

Survey JSON — это полное описание опроса. Хранится в поле `Schema` в базе в формате `jsonb`. Именно этот файл читает `JsonLogicEngine` чтобы решить куда идти после каждого ответа

### Обязательные поля

```json
"required": ["id", "version", "title", "steps", "start"]
```

### Три типа шагов

- `question` — вопрос с ответами
- `branch` — ветвление без вопроса, только логика
- `end` — финальный экран

### Типы ответов

`input`, `textarea`, `select`, `radio`, `checkbox`, `number`, `email`, `date`, `range`, `tel`

### Пример полной схемы

```json
{
  "id": "customer_satisfaction",
  "version": 1,
  "title": { "en": "Survey", "ru": "Опрос" },
  "start": {
    "welcomeText": { "en": "Welcome!", "ru": "Добро пожаловать!" },
    "nextStep": "first_question"
  },
  "steps": {
    "first_question": {
      "type": "question",
      "label": { "en": "Rate us", "ru": "Оцените нас" },
      "answers": [{ "type": "range", "group": "score", "variable": "score", "minValue": 1, "maxValue": 5, "step": 1, "colored": true }],
      "next": {
        "if": [
          [{ "<=": [{ "var": "score" }, 2] }, "low_score"],
          [{ ">=": [{ "var": "score" }, 4] }, "high_score"]
        ],
        "else": "end"
      }
    },
    "low_score":  { "type": "question", "label": { "ru": "Что не понравилось?" }, "answers": [...], "next": "end" },
    "high_score": { "type": "question", "label": { "ru": "Что понравилось?" },    "answers": [...], "next": "end" },
    "end":        { "type": "end", "result": "completed" }
  },
  "variables": {
    "score": { "type": "number", "minimum": 1, "maximum": 5, "required": true }
  },
  "timeouts": { "inactivitySec": 3600 }
}
```

Запрос приходит в контроллер -> сервис управляет прохождение -> движок читает JSON и решает куда идти -> и все хранится в базе 
