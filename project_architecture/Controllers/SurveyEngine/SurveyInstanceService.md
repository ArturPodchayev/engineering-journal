## SurveyInstanceService 

# Является основным файлом (на данный момент) который мне удалось обнаружить. В этом скрипте прописана бизнес логика, старт опроса, его завершение и выдача вознаграждения 

# Именно в этом скрипте прописан метод StartSurvey, который отвечает за несколько тасков:
1. Проверяет авторизован ли пользователь 
2. Проверяет, доступен ли опрос для КОНКРЕТНОГО пользователя
3. Проверка и создание инстансов
4. Возращает стартовый экран + первый вопрос


## SubmitAnswerAsync - Метод который отвечает на вопрос что происходит после того, как пользователь отвечает на вопрос 

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
// объединяем Vars и Result в один объект для движка правил
var combinedData = MergeVarsAndResultForRuleEngine(updatedVars, instance.Result);

// спрашиваем JsonLogicEngine — куда идти дальше?
var nextStep = await _ruleEngine.GetNextStepAsync(schemaDoc, request.StepId, combinedData);

// переводим указатель на следующий шаг
instance.Pointer = nextStep;
```

```csharp
// проверяем финальный ли шаг — три варианта: "end", "complete", или type: "end" в схеме
var isEndStep = nextStep == "end" || nextStep == "complete" || IsEndTypeStep(schemaDoc, nextStep);
if (isEndStep)
    instance.Status = SurveyInstanceEntity.STATUS_COMPLETED;
```
## Методы завершения — все три похожи по структуре

Все три делают одно: достают инстанс → меняют статус → создают событие → сохраняют в базу.

Разница только в причине:

| Метод | Кто вызывает | Статус |
|-------|-------------|--------|
| `CompleteAsync` | пользователь сам завершил | `Completed` |
| `RejectAsync` | пользователь отменил | `Rejected` |
| `ProcessTimeoutAsync` | фоновая задача (30 мин бездействия) | `Expired` |

## Вспомогательные методы

- `GetCachedDefinitionAsync` — берёт схему из кэша или из базы
- `ExtractInitialStepFromSchema` — находит первый шаг из `start.nextStep`
- `ExtractStepFromSchema` — находит конкретный шаг по ID
- `MergeAnswerIntoResult` — сохраняет ответ с меткой времени и текстом вопроса
- `MergeVariablesForBranching` — обновляет переменные для ветвления
- `FormatAnswerValues` — форматирует даты из JS формата в C#
- `IsEndTypeStep` — проверяет является ли шаг финальным

## После завершения опроса

```
SurveyCompletedEvent → очередь
        ↓
OnboardingSurveyCompletedHandler  — если онбординг
WalletSurveyCompletedHandler      — начисление монет
SegmentSurveyCompletedHandler     — обновление квоты сегмента
```
Если в крации: SurveyInstanceService — это сервис который управляет жизненным циклом прохождения опроса

Когда пользователь стартует опрос — сервис проверяет авторизацию, проверяет доступность опроса для конкретного пользователя, создаёт SurveyInstance в базе и устанавливает Pointer на первый шаг из схемы

Когда пользователь отвечает — сервис валидирует что ответ пришёл именно на текущий шаг (через Pointer), защищается от дублей через idempotencyKey, сохраняет ответ в Result, обновляет переменные в Vars для ветвления, и передаёт всё это в JsonLogicEngine который возвращает ID следующего шага. Pointer обновляется на этот шаг

Если следующий шаг финальный — статус инстанса меняется на Completed и публикуется SurveyCompletedEvent в очередь. Обработчики этого события занимаются начислением монет, онбордингом и обновлением сегментом
