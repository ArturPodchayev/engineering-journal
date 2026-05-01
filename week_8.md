# Week 8: 20 – 24 апреля 2026

**в кратце:** восьмая неделя — дебаггинг 401 на ReportTemplate, sql баг в GeoDashboard, реализация и тест фичи временной авторизации (temp token) на клиенте, бэке и в админке и фокус на изучении middleware

---

## над чем работал?

**этап 1 — баг ReportTemplate 401**

- словил 401 на `report/ReportTemplate/paginated` при открытии модуля отчётов
- пробовал добавить доступ роли Administrator через UI — модалка зависала на "Sending" и не сохраняла
- добавил доступ вручную через Swagger (`ReportTemplateController - GET Execute/{Code}/Pagination`)
- выяснил что нужен именно `Paginated` эндпоинт, а не обычный `Get Pagination`

**этап 2 — sql баг в GeoDashboardPage**

- нашёл баг в sql запросе — фильтрация шла по `ua."Name"::jsonb ->> 'ru' = 'country'`, но в базе значение хранится как `{"ru":"Страна","en":"Country"}`
- в `UserAttributeValues` не было тестовых данных, скрипт `seed-test-users.sh` падал с `column "Language" does not exist`
- Роман пояснил что sql был временным для проверки дашборда — реальная логика будет привязана к UUID конкретного UserAttribute

**этап 3 — фича временной авторизации (temp token)**

- проблема: на клиенте `?token=` вообще не отрабатывал, в админке давал доступ к одному запросу и выбрасывал пользователя
- корневая причина: `telegramAuthInterceptor` шёл первым, вешал `Authorization` хедер, `temporaryTokenInterceptor` видел его и делал ранний выход — обмен токена на JWT не происходил
- на бэке перевёл эндпоинт `AuthorizationController` с `[HttpPost]` на `[HttpGet] + [FromQuery]`
- на фронте в админке: перехват `?token=` в `app.component.ts`, метод `authorizeWithToken` в `users.service.ts`, логика обмена в `is-authorized.guard.ts`, убрал `temporaryTokenInterceptor` из `authProviders.config.ts`
- протестировал три сценария: обмен токена на JWT, блокировку повторного входа после истечения, работу пользователя в рамках роли
- фича в админке готова и протестирована локально, PR не открыт

---

## технические заметки

порядок интерсепторов в Angular:

```typescript
// неправильно — telegramAuthInterceptor первым вешает Authorization
// temporaryTokenInterceptor видит хедер и делает early return

// правильно — temporaryTokenInterceptor идёт первым
// при наличии ?token= явно удаляем Authorization от предыдущего интерсептора
```

jsonb фильтрация в PostgreSQL:

```sql
-- неправильно
ua."Name"::jsonb ->> 'ru' = 'country'

-- правильно (в базе: {"ru":"Страна","en":"Country"})
ua."Name"::jsonb ->> 'ru' = 'Страна'
```

---

## что я выучил?

порядок интерсепторов в Angular критичен — если несколько интерсепторов работают с одним хедером, первый выигрывает и остальные могут не отработать

`jsonb` в PostgreSQL чувствителен к значениям — фильтрация по неправильному значению вернёт пустой результат без ошибки, что сложно дебажить

эндпоинты для обмена токена лучше делать `GET + query param` — проще вызвать по ссылке без тела запроса

---

**энергия: 7/10**

**оценка недели: 6/10**
