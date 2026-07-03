# неделя — 29 июня – 3 июля 2026

**одним предложением:** дописал integration-plan для fingerprint-сервиса, поднял рабочий стенд на бэке и фронте , застрял на 403 при пуше — жду доступ от тимлида

---

## над чем работал

**02-integration-plan.md — архитектура fingerprint/anti-fraud**

добивал документ по системному дизайну, который начал раньше тимлид спрашивал конкретно — как бэк принимает данные , как обрабатывает, куда складывает, синк или асинк вызываются geolite2/asn/ip2proxy

прошел через это сократовским методом сам додумался до middleware → controller → job , вместо того чтобы контроллер сам дергал geolite2 и ждал ответа контроллер не должен быть комбайном

итоговая архитектура:

```
POST /api/fingerprint
    └── Middleware (auth, rate limit, validate)
    └── FingerprintController
            ├── сохранить сырые данные в БД
            ├── dispatch(ProcessFingerprintJob)
            └── return 200 OK

ProcessFingerprintJob (асинхронно , в очереди)
            ├── GeoLite2 City → страна, город
            ├── GeoLite2 ASN  → провайдер, ASN
            ├── IP2Proxy      → тип IP (vpn/proxy)
            └── результат → fingerprint_geo
```

документ оформлен в `technologies-learning/02-integration-plan.md` , схема продублирована в miro

**реализация бэка (laravel , репо surveys)**

поднял проект через docker (`docker compose -f dev_build/docker-compose.dev.yml`) , прогнал миграции , все окей

сделал:
- модели `Fingerprint` и `FingerprintGeo`
- `FingerprintController`
- `ProcessFingerprintJob` — асинхронное обогащение через geolite2 city/asn + ip2proxy
- роут `POST /api/v2/fingerprint`
- таблицы `fingerprints` и `fingerprint_geo`

все протестировал через `php artisan tinker` — руками прогнал job и проверил что geo-данные реально складываются

**реализация фронта (angular 19, репо web-app-opinion)**

собрал три сервиса по плану:
- `FingerprintCollectorService` — собирает device-данные (fingerprintjs oss, user-agent, language, timezone, screen resolution, device pixel ratio, webgl renderer)
- `FingerprintEventService` — шина на rxjs subject, слушает триггеры
- `FingerprintApiService` — шлёт payload на бэк

подключил через `provideAppInitializer` — `APP_INITIALIZER` в angular 19 задеприкейтили , теперь так

билд прошел без ошибок

**триггеры отправки зафиксировал финально:**
- `user_registered`
- `user_logged_in`
- `survey_started`
- `survey_completed`
- `payment_initiated`
- `personal_data_changed`

`answer_given` намеренно не включил — иначе на каждый вопрос в опросе будет свой запрос , а это уже не fingerprint, а спам (imho)

**застрял на пуше**

попробовал запушить ветку в `ngroup-insight/surveys` — `403: Write access to repository not granted`

проверил через `git remote -v` — путь верный, проблема не в этом зашёл в settings репо, там только read-only

то же самое с `web-app-opinion` — тоже 403

написал тимлиду , жду пока добавит с правами на запись пока код лежит закоммиченным локально , ниче не потеряно

---

## техническое решение недели

middleware → controller → job — это не просто способ разложить код, это способ не блокировать ответ клиенту пока идёт медленная работа (geolite2/ip2proxy лукапы) контроллер обязан ответить быстро , а обогащение может подождать своей очереди

когда решение поначалу кажется очевидным (например "просто дернуть geolite2 в контроллере") — стоит спросить себя что будет, если этот вызов подвиснет или упадёт если ответ "весь запрос упадёт вместе с ним" — это сигнал выносить в job

---

## что выучил

`provideAppInitializer` — не косметическая замена `APP_INITIALIZER`, а обязательная миграция для angular 19, если хочешь без deprecation warning

асинхронная обработка — это не про скорость самого geolite2-лукапа, а про то чтобы пользователь не ждал пока бэк сходит в три внешних источника прежде чем ответить 200

read-only доступ в репозитории — не блокер для работы, а просто повод писать код локально и не бояться коммитить пока ждёшь доступ

---

## итоги недели 🏆

- закрыт `02-integration-plan.md` — архитектуру накидал , мне очень понравилось 
- поднят рабочий стенд: бэк (laravel) + фронт (angular) для fingerprint-сервиса
- протестирована связка через `php artisan tinker`
- миро-схема синхронизирована с документом
- обнаружена проблема с доступом — заблокирован на пуше (временно)
- начал пить ежовик, мне очень нравится , чувствую что мыслю лучше
---

**энергия: 10/10**
**оценка недели: 8/10**
