# Week 3 — 16–20 марта 2026

**One sentence that captures this week:** Третья неделя — впервые самостоятельно нашёл баг, пофиксил его и закрыл первый PR

---

## What I built / worked on

**Этап 1 — i18n задача (переводы):**
- Получил задачу добавить недостающие ключи переводов для трёх языков (en/ru/uz)
- Прошёлся по трём языковым файлам и нашёл все отсутствующие ключи
- Добавил переводы для:
  - `Breadcrumbs.DynamicBlocks` — раздел "Динамические блоки"
  - `TableView.LdapOrganizationEntity` — полный блок с полями name, description, code, defaultRoleId, id, username, ip, password
  - `TableView.RoleEntity.defaultForCode` — недостающий ключ для колонки в таблице ролей
  - `TableView.SurveyLaunchEntity` и `DataEdit.SurveyLaunchEntity` — добавлены useLatestVersion и status
- Создал ветку, закинул PR — смёрджили без замечаний 

**Этап 2 — Survey Definition (разбор архитектуры):**
- Разобрал полный путь опроса от канваса до базы данных — и туда и обратно
- Прошёлся по фронту: `SurveyEditorManagerService`, `StepPropertiesPanel`, `survey-editor.component.ts`, `survey-transformation.service.ts`, `survey-defenition.service.ts`
- Прошёлся по бэку: `SurveyDefinitionController`, `SurveyDefinitionService`, `SurveyDefinitionRepository`, `BaseEntityService`
- Написал подробный markdown документ на GitHub

---

## Technical decision of the week

**Decision:** Читать код сверху вниз по слоям — фронт → контроллер → сервис → репозиторий → БД

**Why:** Когда понимаешь как данные движутся через всю систему, любой баг или новая фича сразу ложится на понятную карту

**What I'd do differently:** Начинать с DTO — они как контракт между слоями, сразу видно что летит туда и обратно

---

## Key concepts I learned this week

- **BehaviorSubject** — реактивное хранилище состояния. Хранит последнее значение и оповещает всех подписчиков при изменении. Весь канвас редактора строится на этом
- **Template Method pattern** — `BaseEntityService` определяет скелет алгоритма (маппинг → хук → репозиторий → сохранение), наследники переопределяют только свою логику через `override`
- **jsonb в PostgreSQL** — Schema и VisualSchema хранятся не как строки а как нативный JSON, можно делать запросы внутрь документа
- **[NotMapped]** — атрибут EF Core, поле есть в Entity но не хранится в БД. `ActiveInstanceId` вычисляется на лету
- **Инвалидация кэша** — после сохранения опроса сервис сам чистит кэш, чтобы следующий GET не вернул устаревшие данные

---

## AI usage this week

- **Used AI correctly:** Разбор Survey Definition — читали файлы вместе, я задавал вопросы и разбирался в логике
- **Used AI as a crutch:** Часть markdown документа написал AI — надо писать самому и просить только проверить

---

## Questions I asked the team (and what I learned)

- Q: Где лежат языковые файлы и как добавить ключ? → A: Показали структуру i18n папки, дальше разобрался сам

---

## Wins this week 

- Первый реальный баг пофиксил — и он реально пофиксился
- Первый PR — смёрджили без замечаний
- Разобрал полную архитектуру Survey Definition от канваса до БД

---

**Energy level this week: 10/10** — неделя когда понимаешь что реально что-то делаешь, а не просто смотришь
