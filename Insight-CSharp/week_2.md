# Week 2 — 10–14 марта 2026

**One sentence that captures this week:** Вторая неделя — провалидировал баг с radio button, нашёл что проблема на фронте, и разобрал архитектуру test-survey модуля.

---

## What I built / worked on

**Этап 1 (вторник-среда) — Radio Button Bug:**

- Подтянул изменения разраба из гита, пересобрал контейнеры
- Создал тестовый опрос с двумя radio button с разными группами (`group_1`, `group_2`)
- Прошёл опрос, проверил результат через UI и через Network в DevTools
- Нашёл что фикс разраба работает частично — если выбрать только один вариант всё окей, но если выбрать оба — сохраняется только последний
- Прошёлся по бэкенду: `SubmitAnswerAsync`, `NormalizeAnswerValue`, `MergeAnswerIntoResult` — бэк чистый
- Нашёл баг в `survey-runner.component.ts` — в методе `onSubmitAnswer` одна строка `answerValue = value` перезаписывает ответ вместо `answerValue[groupName] = value`
- Отписал разрабу — баг на фронте, бэк вопросов нет

**Этап 2 (четверг-пятница) — test-survey:**

- Изучил архитектуру `test-survey` модуля — тестовый прогон опроса
- Разобрал файлы: `test-survey.component.ts`, `survey.service.ts`, `survey.models.ts`, `question-base.component.ts`, `multiple-choice-question.component.ts`
- Написал markdown документ на GitHub

---

## Technical decision of the week

**Decision:** Проверять баги через Network DevTools до чтения кода  
**Why:** Сразу видно что летит на бэк — если данные уже неправильные в Payload, копать бэк смысла нет  
**What I'd do differently:** Сразу открывать DevTools при получении баг-репорта, не тратить время на догадки

---

## Key concepts I learned this week

- **BehaviorSubject** — реактивное хранилище состояния в Angular. Хранит последнее значение и оповещает всех подписчиков при изменении
- **Template Method pattern** — `QuestionBaseComponent` определяет скелет алгоритма, наследники (`SingleChoice`, `MultipleChoice`) заполняют конкретные шаги
- **Abstract class в Angular** — базовый компонент с абстрактными методами, наследники обязаны их реализовать
- **Grouped radio buttons** — radio button с разными `group` не исключают друг друга, можно выбрать оба. Это не баг HTML, а можно сказать что это ожидаемое поведение

---

## AI usage this week

- **Used AI correctly:** Диагностика бага — прошёлся по методам бэка вместе, нашли что проблема не там
- **Used AI correctly:** Изучение незнакомого кода — разбирали `test-survey` файл за файлом
- **Used AI as a crutch:** Markdown документ по `test-survey` написал AI, не я

---

## Questions I asked the team (and what I learned)

- Q: Баг с radio button — бэк или фронт? → A: "баг на фронте"

---

## What I'll do differently next week

- При получении баг-репорта сразу открывать Network DevTools
**Energy level this week: 9/10**  
Неплохая неделя, уже неплохо так освоился в офисе, заряжен
