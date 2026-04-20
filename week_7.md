# Week 7: 13 – 17 апреля 2026

**В кратце:** седьмая неделя — смержили ветки, разрешили конфликты в бэке, потестил роль Administrator, поизучал архитектуру, закрыл большой PR по i18n переводам для 3 языков

---

## над чем работал?

**Этап 1 — Мерж и конфликты в бэке:**

- обновился по всем мастер веткам (insight, MicroservicesAdmin, Microservices)
- разрешил конфликты мержа в SurveyInstanceController, SurveyInstanceService, appsettings.json
- пересобрал surveysenginemicroservice в докере
- изучил роль Administrator на `SegmentConditionController - GET attribute-values/{userAttributeId}`

**Этап 2 — PR #16 i18n переводы (InsightAdmin):**

- прошелся по всем страницам AdminPanel и собрал все WARN ключи без переводов
- добавил недостающие ключи для ru/en/uz: DictionaryEntity, DictionaryEntryEntity, BotMessageEntity, SurveyLaunchEntity (useLatestVersion, sendActivationNotifications, status, квоты), LdapOrganizationEntity, UserEntity (firstName, lastName, balance, passedOnboarding), RoleEntity (defaultForCode), Breadcrumbs (Role, User, DynamicBlocks, BotMessages, Dictionaries), ErrorTips.Required
- пофиксил дублирующиеся блоки `TabLabels` и `SurveyLaunchEntity` в DataEdit
- словил merge conflict в пятницу вечером — не сделал `git pull` перед работой
- починил через перезапись файлов чистыми версиями
- PR #16 ветка `translation/fix-missing-keys-2` — Ready to merge

---

## Технические заметки

**Дублирующиеся ключи в JSON:**

```json
// Оба блока называются TabLabels — второй перезапишет первый
"TabLabels": { "RoleEntity": "Роли" },
...
"TabLabels": { "SegmentEntity": "Сегменты" }
// Итог — первый блок игнорируется полностью
```

лучшее решение — всегда сливать все ключи в один блок

**Merge conflict — почему возник:**

```
// master ушел вперед пока работал над веткой
// не сделал git pull перед началом → ветка стартовала со старой версии файлов
// Git видит изменения в одних строках с обеих сторон -> вставляет маркеры

<<<<<<< HEAD
"Role": "Роль"
=======
"Roles": "Роли"
>>>>>>> origin/master
```

решение — перезаписать файлы чистыми версиями, `git add`, `git commit`, `git push`

**Почему `git checkout --theirs` не сработал:**

файлы уже были застейджены через `git add` — команда не может перезаписать застейджированный файл и нужно было сначала `git restore --staged <file>` либо просто перезаписать файл вручную

---

## Что я выучил?

JSON не поддерживает дублирующиеся ключи — последний блок с одинаковым именем всегда перезаписывает предыдущий, первый молча игнорируется

`git checkout --theirs` не работает на застейджированных файлах

---

**Энергия:** 7/10

**Оценка недели:** 6/10
