## Temp Token Authorization — Client (insight-client)

### суть задачи
юзер получает ссылку с временным токеном (?token=...) и должен попасть в приложение в рамках своей роли, без регистрации через Telegram

### как это работает?
1. пользователь переходит по ссылке (http://localhost:4201?token=токен)
2. app.component.ts читает токен из URL в ngOnInit
3. вызывается GET /identity/Authorization/authorizeWithToken?temporaryToken=...
4. бэк возвращает JWT и он сохраняется в sessionStorage
5. temporaryTokenInterceptor подставляет JWT как Authorization: биррер на последующие запросы

### почему именно так?
- temporaryTokenInterceptor нельзя оставлять в цепочке — он добавляет ?token= ко всем запросам включая сам authorizeWithToken, что ломает обмен
- telegramAuthInterceptor вешает tma хедер на все запросы — нужно исключение для authorizeWithToken иначе бэк получает авторизованный запрос на Unauthorized эндпоинт и падает с 500
- + в base64 токене декодируется браузером как пробел — нужно читать URL через window.location.search.replace(/\+/g, '%2B') перед передачей в URLSearchParams

### ключевые файлы
- src/app/app.component.ts — точка входа, читает токен и вызывает authorizeWithToken
- src/app/services/user-service/user.service.ts — метод authorizeWithToken с правильным энкодингом
- src/app/interceptors/telegram-auth.interceptor.ts — исключение для authorizeWithToken
- src/app/interceptors/temporary-token.interceptor.ts — подставляет JWT из sessionStorage в запросы
- src/app/app.config.ts — temporaryTokenInterceptor убран из цепочки

### бэк
эндпоинт authorizeWithToken должен быть HttpGet + FromQuery — TemporaryTokenAuthMiddleware в Ocelot сам делает GET запрос для валидации токена, если эндпоинт POST — middleware получает 405 и блокирует запрос

### что осталось?
После успешного authorizeWithToken и редиректа на /main — surveysengine/SurveyUser/currentUser падает с 500 потому что Telegram перезаписывает JWT своим tma токеном и получается, это уже отдельная задача на стороне surveysengine
