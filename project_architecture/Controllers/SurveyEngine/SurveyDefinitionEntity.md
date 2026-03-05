# SurveyDefinitionEntity.cs

`SurveyDefinitionEntity` — это модель данных которая описывает как опрос хранится в базе данных. Каждое поле = колонка в таблице `surveyengine."SurveyDefinitions"` 

## Индексы

```csharp
[ComplexIndex(nameof(Id), nameof(Version))]
[ComplexIndex(nameof(Status))]
[ComplexIndex(nameof(OwnerSurveyUserId))]
```
Индексы — это как оглавление в книге. Без них база читает всё с начала чтобы найти нужную запись. С ними — сразу открывает нужную страницу. Эти три говорят базе: "часто ищем по `Id+Version`, `Status` и `OwnerSurveyUserId` — сделай быстрый доступ"


## Ключевые поля

```csharp
[Required]
[Column(TypeName = "jsonb")]
public JsonDocument Schema { get; set; } = default!;
```

Самое важное поле. `jsonb` — это JSON прямо в PostgreSQL, можно делать запросы внутрь JSON. `[Required]` — без схемы опрос создать нельзя. Это и есть тот самый Survey JSON который читает `JsonLogicEngine`
