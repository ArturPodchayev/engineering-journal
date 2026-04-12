# Week 6: 6 – 11 апреля 2026

**В кратце:** шестая неделя — закрыл два PR, пофиксил баг с дублированием сайдбара, разобрал файловый микросервис и схему обмена файлами через RabbitMQ

---

## Над чем работал

**Этап 1 — PR #9 checkbox_exclusive (InsightAdmin):**

- Реализовал исключающий чекбокс — при выборе сбрасывает все остальные в группе
- Затронул 6 файлов: enum в SurveySchemaTypes и AnswerInputType, step-properties-panel, survey-runner .ts и .html, survey-definition.schema
- Урок: не лезть в бэк без уточнения scope у Романа — задача была только на фронт
- PR #9 ветка `feature/checkbox-exclusive-v2` — Ready to merge, ждёт Романа

**Этап 2 — Баг с дублированием сайдбара (MicroservicesAdmin PR #12):**

- Роман описал баг: при возврате из конструктора опросов пункты меню дублируются с каждым переходом
- Нашёл причину в `side-nav.component.ts` — `var routes = ROUTES` это не копия массива а указатель на оригинал, каждый `push` мутировал глобальный массив
- Фикс — одна строка: заменил на `let routes = [...ROUTES, ...registeredRoutes]`
- По ходу заменил `var` на `const`/`let` где уместно
- Запушил PR #12 на ветку `feature/course-constructor`

**Этап 3 — Изучение FilesMicroservice:**

- Разобрал структуру файлового микросервиса — контроллеры, сервисы, консьюмеры
- Изучил библиотеку `Microservices.Shared.FilesMicroservice.RPC` — как другие микросервисы обращаются к файлам через Rabbit
- Разобрал два маппера — сохранение файла и получение файла
- Написал документ в журнал

---

## Технические заметки

**Мутация глобального массива:**

```typescript
// было — routes указывает на тот же массив в памяти
var routes = ROUTES;
routes.push(...registeredRoutes); // мутирует оригинал

// стало — создаётся свежая копия каждый раз
let routes = [...ROUTES, ...registeredRoutes];
```

**RPC паттерн через RabbitMQ:**

Микросервисы не ходят в FilesMicroservice по HTTP — только через очередь Rabbit. Это развязывает зависимости — можно менять реализацию внутри не трогая потребителей

**Паттерн Strategy + Factory в FilesMicroservice:**

```csharp
switch (item.FileType)
{
    case FileType.File:
        result = await _factory.GetStrategy<FileEntity>().AddAsync(item);
        break;
    case FileType.ChunkedFile:
        result = await _factory.GetStrategy<ChunkedFileEntity>().AddAsync(item);
        break;
}
```

Добавить новый тип файла = добавить новую стратегию, ничего не ломая

---

## Что я выучил

`[...array]` — spread создаёт поверхностную копию, оригинал не трогается

`var` vs `const`/`let` — в современном TS `var` не используется: `const` если не переприсваивается, `let` если переприсваивается
---

## Мини-победы

- Нашёл и пофиксил баг с сайдбаром за один день
- PR #9 checkbox_exclusive — Ready to merge без конфликтов
- Разобрал целый микросервис самостоятельно

---

**Энергия:** 9/10

**Оценка недели:** 8/10
