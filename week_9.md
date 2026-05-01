# Week 9: 27 апреля – 1 мая 2026

**в кратце:** девятая неделя — довел temp token на клиенте до рабочего состояния, разобрался с приоритизацией токенов на бэке, еще глубже углубился в архитектуру, протестировал и починил CI/CD пайплайн на insight-rc

---

## над чем работал?

**этап 1 — temp token на insight-client**

- `authorizeWithToken` работает, ловим 200 и получаем JWT
- проблема: `auth.guard` на `/main` вызывал `surveysengine/SurveyUser/currentUser` без телеграм токена и получал CORS/500
- разобрался с корневой причиной — `telegramAuthInterceptor` перезаписывал temp JWT своим `tma` токеном
- итоговая схема: `app.component.ts` читает токен из URL в `ngOnInit`, `temporaryTokenInterceptor` вешает его как `Authorization: tmp <токен>`, бэк возвращает JWT, он сохраняется в `sessionStorage`, последующие запросы идут с `Authorization: Bearer`
- баг с `+` в base64 — декодируется браузером как пробел, фикс: читать URL через `window.location.search.replace(/\+/g, '%2B')` перед `URLSearchParams`
- выяснил что проблема с `currentUser` — это отдельная задача на стороне `surveysengine`, не в рамках этой фичи

**этап 2 — разобрался с архитектурой токенов**

- temp token ориентирован на веб-версию, tma — только для Telegram WebApp
- `authorizeWithToken` — внутренний метод для генерации JWT, в будущем middleware перепишут на RabbitMQ вместо HTTP запроса
- эндпоинт `authorizeWithToken` обязательно должен быть `[HttpGet] + [FromQuery]` — `TemporaryTokenAuthMiddleware` в Ocelot делает GET запрос, при POST получает 405 и блокирует

**этап 3 — тест CI/CD пайплайна (insight-rc)**

- переключился на ветку `insight-rc`, изучил архитектуру: workflow → self-hosted раннер → `deploy-microservices-rc.sh` → `docker compose up --build`
- провел локальный smoke-тест через `bash docker-projects/insight/scripts/ci/deploy-microservices-rc.sh`
- нашел и починил 3 бага в `docker-compose.engine.yml`:
  - дублирующийся блок `reportsmicroservice` — удалил второй
  - дублирующийся `restart: unless-stopped` в `surveysenginemicroservice` — удалил лишний
  - healthcheck в `filesmicroservice` использовал `curl` которого нет в .NET minimal образе — убрал
- запушил фикс в `insight-rc`, получил PAT от данияра, поднял раннер через `docker-compose.runner.yml`
- полный цикл прошел: пуш -> GitHub Actions → раннер подхватил -> все 6 сервисов пересобрались -> `deploy finished` 

---

## технические заметки

base64 в query string:

```typescript
// + в base64 декодируется браузером как пробел
// правильно читать так:
const search = window.location.search.replace(/\+/g, '%2B')
const params = new URLSearchParams(search)
const token = params.get('token')
```

приоритизация токенов на бэке:

```
tmp <токен>   — временная авторизация (веб)
ey<JWT>       — обычная авторизация
tma <токен>   — Telegram WebApp
```

CI/CD схема:

```
push в insight-rc → GitHub Actions создает job → self-hosted раннер подхватывает
→ git pull insight-rc → docker compose up -d --build → сервисы пересобраны
```

---

## что я выучил?

middleware в Ocelot делает собственный HTTP запрос при валидации токена — если эндпоинт POST, middleware получает 405 и блокирует весь запрос

приоритизация интерсепторов в Angular решает кто "победит" при конфликте хедеров — порядок в `app.config.ts` имеет значение

`healthcheck` в docker compose требует наличия `curl` или `wget` в образе — в .NET minimal образах их нет по умолчанию

---

**энергия: 9/10**

**оценка недели: 8/10**
