# fingerprint и anti-fraud — integration plan

## цель
наша цель - собирать fingerprint данные пользователя по ключевым событиям , отправлять их на бэк для хранения и асинхронного обогащения через GeoLite2/ASN/IP2Proxy в целях антрифрод анализа

## архитектура фронтенда (ангуляр)
### сервисы
**FingerprintCollectorService** — собирает данные устройства:
- FingerprintJS OSS (visitorId)
- user-agent , Language , Timezone
- Screen Resolution , Device Pixel Ratio
- WebGL Renderer

**FingerprintEventService** — шина событий
инициализируется через `APP_INITIALIZER`
слушает события , затем вызывает Collector и передает в api

**FingerprintApiService** — отправляет payload на бэк (`POST /api/fingerprint`)
## триггеры отправки

- `user_registered`
- `user_logged_in`
- `survey_started`
- `survey_completed`
- `payment_initiated`
- `personal_data_changed` 

## архитектура бэка (laravel)

### слои обработки

**Middleware** — auth , rate limit , validate  (http://github.com/ArturPodchayev/engineering-journal/blob/main/Insight-CSharp/Middleware/middleware_how_it_works.md)
**FingerprintController** — принимает запрос , сохраняет сырые данные в `fingerprints` , кидает Job в очередь , возвращает `200`
**ProcessFingerprintJob** — асинхронная обработка:
- GeoLite2 City — страна и город
- GeoLite2 ASN — провайдер , ASN
- IP2Proxy — тип IP (vpn / proxy)
- результат сохраняется в `fingerprint_geo`

### бд

**`fingerprints`** — сырые данные с фронта (visitor_id, user_agent , ip , event_type)

**`fingerprint_geo`** — данные от job (country , city, proxy_type)

### схема системного дизайна - https://miro.com/app/board/uXjVHAcRzgc=/?share_link_id=824914227896 
