# Survey JSON (схема опроса)

Survey JSON — это полное описание опроса. Хранится в поле `Schema` таблицы `surveyengine."SurveyDefinitions"` в формате `jsonb`

Именно этот файл читает `JsonLogicEngine` чтобы решить куда идти после каждого ответа. Именно отсюда `SurveyInstanceService` достаёт первый шаг, текущий вопрос и правила ветвления

Этот конкретный файл — это **JSON Schema**, то есть не сам опрос, а правила валидации по которым проверяется любой опрос в системе

## Обязательные поля

```json
"required": ["id", "version", "title", "steps", "start"]
```

Без любого из них опрос создать нельзя:

- `id` — машиночитаемый идентификатор, например `survey.customer_eligibility`
- `version` — версия опроса, неизменна после публикации
- `title` — локализованное название `{ "en": "...", "ru": "..." }`
- `steps` — все шаги опроса
- `start` — стартовый экран с `nextStep` — откуда начинать

## Структура опроса

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
    "first_question": { "type": "question", ... },
    "branch_step":    { "type": "branch", ... },
    "end":            { "type": "end", ... }
  },
  "variables": { ... },
  "timeouts": { "inactivitySec": 3600 }
}
```

## Три типа шагов

**`question`** — вопрос с ответами, пользователь что-то выбирает или вводит:
```json
{
  "type": "question",
  "label": { "en": "How old are you?", "ru": "Сколько вам лет?" },
  "answers": [...],
  "next": "next_step"
}
```

**`branch`** — ветвление без вопроса, только логика:
```json
{
  "type": "branch",
  "next": {
    "if": [[{ ">=": [{ "var": "age" }, 18] }, "adult_step"]],
    "else": "minor_step"
  }
}
```

**`end`** — финальный экран, опрос завершён:
```json
{
  "type": "end",
  "result": "completed"
}
```

## Типы ответов (виджеты)

| Тип | Что это |
|-----|---------|
| `input` | текстовое поле |
| `textarea` | большое текстовое поле |
| `select` | выпадающий список с `possibleVariants` |
| `radio` | выбор одного из нескольких (кружочки) |
| `checkbox` | да/нет галочка |
| `number` | числовое поле |
| `email` | поле email |
| `date` | дата/время с форматом |
| `range` | ползунок с `minValue`, `maxValue`, `step` |
| `tel` | телефон с форматом |

## Правила ветвления — связь с JsonLogicEngine

Каждое правило это массив из двух элементов: `[условие, следующий шаг]`:

```json
"next": {
  "if": [
    [{ "==": [{ "var": "score" }, "1"] }, "low_step"],
    [{ ">=": [{ "var": "score" }, "4"] }, "high_step"]
  ],
  "else": "feedback"
}
```
