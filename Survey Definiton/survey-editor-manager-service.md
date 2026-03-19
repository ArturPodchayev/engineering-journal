# SurveyEditorManagerService — менеджер состояния редактора
## Что это?

Центральный сервис редактора опросов. Хранит всё состояние канваса — шаги, соединения, переменные
Все остальные компоненты редактора работают через него

## Хранение состояния

```typescript
private _editorState = new BehaviorSubject<SurveyEditorState>(new SurveyEditorState());
public readonly editorState$: Observable<SurveyEditorState> = this._editorState.asObservable();
```
Состояние хранится в `BehaviorSubject`. Все компоненты подписываются на `editorState$` и автоматически получают обновления когда что-то меняется
Геттер и сеттер:
- `get editorState()` — просто возвращает текущее состояние
- `set editorState(state)` — перед сохранением синхронизирует `stepCounter` чтобы новые шаги не получили уже существующий ID, потом стреляет в `BehaviorSubject`
---
## Основной функционал

### Создание нового опроса
```typescript
createNewSurvey()
```
Создаёт пустое состояние и сразу добавляет START шаг на позицию (100, 100)
Сбрасывает счётчик шагов
---

### Добавление шага на канвас
```typescript
addStep(stepType, x, y)
```
В зависимости от типа вызывает нужный приватный метод:
- `createStartStep()` — stepId всегда `"start"`, добавляет дефолтный welcomeText на всех языках
- `createQuestionStep()` — stepId = `step_N`, пустой массив ответов
- `createBranchStep()` — stepId = `branch_N`, пустые правила `{ if: [], else: "" }`
- `createEndStep()` — stepId = `end_N`, result = `"completed"`

После создания проверяет через `ensureUniqueStepId()` — нет ли уже шага с таким ID
Если есть — добавляет суффикс пока ID не станет уникальным

---

### Удаление шага
```typescript
removeStep(stepId)
```
Три действия сразу:
1. Удаляет шаг из массива
2. Удаляет все соединения где этот шаг был source или target
3. Если удалили шаг на который указывал START — очищает `startScreen.nextStep`

---

### Создание соединения между шагами
```typescript
createConnection(sourceStepId, targetStepId)
```
Перед созданием проверяет лимит исходящих соединений для типа шага:
- `END` — лимит 0, соединение не создаётся
- `QUESTION` — лимит 1, если уже есть соединение — удаляет старое и создаёт новое
- `BRANCH` — без лимита, можно создавать сколько угодно

После создания вычисляет `waypoints` — точки входа и выхода стрелки на границах блоков

---

### Перемещение шага
```typescript
updateStepPosition(stepId, x, y)
```
Обновляет координаты шага. Потом перерисовывает все соединения которые к нему подключены — пересчитывает `waypoints` чтобы стрелки следовали за блоком

---

### Счётчик шагов
```typescript
private stepCounter = 0;
```
Каждый новый Question/Branch/End шаг получает ID типа `step_1`, `step_2` и тд. Счётчик растёт

При загрузке существующего опроса — `syncStepCounter()` парсит все существующие ID, находит максимальное число и ставит счётчик на `max + 1`. Так новые шаги никогда не получат уже существующий ID
