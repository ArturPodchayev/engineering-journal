# StepPropertiesPanel — панель свойств шага 

## Что это?
Это правая панель в редакторе опросов. Появляется когда кликаешь на любой шаг на канвасе. Здесь редактируешь всё что связано с шагом — название, ответы, тип ответа, label на разных языках 

## Как получает данные? 

Получает выбранный шаг через `@Input()`:
```typescript
@Input() selectedStep: SurveyStepEntity | null = null;
```
Когда шаг меняется — срабатывает `ngOnChanges()` который вызывает `loadStepData()` и загружает данные шага в форму 
 
Когда что-то изменилось — стреляет событие вверх через `@Output()`:
```typescript
@Output() stepUpdated = new EventEmitter<void>();
@Output() closePanel = new EventEmitter<void>();
```
## Типы шагов
Панель знает о трёх типах шагов и для каждого отдаёт свои данные:
- `getStartData()` — данные START шага (welcomeText, description, buttonText)
- `getQuestionData()` — данные Question шага (answers)
- `getEndData()` — данные END шага (result)
- `getBranchData()` — данные Branch шага (условия)

 
## Работа с ответами (Question шаг)
 
### Добавить ответ
```typescript
addAnswer()
```
Создаёт новый ответ с типом `INPUT` и группой `default`. А Label сразу создаётся пустым для всех языков. 
`variable` — опциональное, пользователь добавляет вручную если это нужно
 
### Удалить ответ
```typescript
removeAnswer(index)
```
Просто удаляет ответ из массива по индексу.
 
### Поменять тип ответа
```typescript
updateAnswerType(index, type)
```
Меняет тип и добавляет type-specific поля:
- `SELECT` → добавляет `possibleVariants: []`
- `DATE` → добавляет `dateType`, `format: 'YYYY-MM-DD'`
- `RANGE` → добавляет `minValue: 0`, `maxValue: 100`, `step: 1`, `colored: false`
- `TEL` → добавляет `phoneFormat` с дефолтным форматом
 
---

## Мультиязычный редактор
 
Почти все текстовые поля поддерживают несколько языков. Для каждого поля есть три метода:
- `open*Editor()` — открывает модалку с полями для каждого языка
- `close*Editor()` — закрывает без сохранения
- `save*Editor()` — сохраняет и закрывает
 
Поля с мультиязычным редактором:
- Label шага
- Label каждого ответа
- WelcomeText (START шаг)
- Description (START шаг)
- ButtonText (START шаг)
 
---
 
## Обновление Step ID
 
```typescript
updateStepId()
```
Меняет ID шага. Важный момент — автоматически обновляет все соединения на канвасе где этот шаг был source или target, 
Ну а если такой ID уже есть — показывает alert и откатывает изменение
---

## Как изменения попадают в канвас?
 
Каждый метод который что-то меняет вызывает `notifyUpdate()`:
```typescript
private notifyUpdate(): void {
    this.editorManager.editorState = this.editorManager.editorState;
    this.stepUpdated.emit();
}
```
Это триггерит `BehaviorSubject` в `SurveyEditorManagerService` — канвас перерисовывается автоматически 
 
