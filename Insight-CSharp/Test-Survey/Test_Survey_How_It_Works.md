# test-survey — как работает тестовый прогон 

# Это специальный режим для тестирования опроса. Не создаёт реальный инстанс для пользователя — просто прогоняет опрос чтобы проверить что всё работает


```
URL /test-survey/:id
    ↓
TestSurveyComponent — берёт surveyId из URL
    ↓
POST /surveys/Attempt/testStart → получаем первый вопрос
    ↓
Показываем вопрос, пользователь отвечает
    ↓
POST /surveys/Attempt/answer → получаем следующий вопрос
    ↓
isFinished: true → конец
```


## Файлы которые я смотрел

### test-survey.component.ts
Главный компонент. Берёт `surveyId` из URL, запускает опрос, подписывается на состояние через `surveyState$`. При каждом новом вопросе пересоздаёт форму


### survey.models.ts
Основные интерфейсы:
- `SurveyAttemptAnswerRequestDTO` — то что летит на бэк при ответе: `surveyAttemptId`, `questionId`, `questionsAnswers`
- `SurveyAttemptAnswerResponseDTO` — то что приходит с бэка: следующий вопрос или `isFinished: true`
- `QuestionResponseDTO` — вопрос с локализованным текстом и массивом ответов

### question-base.component.ts
Абстрактный базовый класс для компонентов вопросов. Получает `question` и `formGroup` через `@Input`. Определяет абстрактные методы которые обязаны реализовать наследники: `initializeComponent()`, `isValid()`, `getSelectedAnswers()`

### multiple-choice-question.component.ts
Наследует `QuestionBaseComponent`. Отвечает за чекбоксы. Хранит два списка — `selectedAnswerIds` и `selectedLabelValues`. При каждом клике обновляет оба списка через `updateSelectedAnswers()`

# Флоу: старт → вопрос → ответ → следующий вопрос → конец
