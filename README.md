# WeChat sandbox service account: PHP webhook and test panel

Минимальный PHP-проект для тестового аккаунта WeChat Official Accounts Platform. Он показывает два потока:

1. **Ответ на входящее сообщение.** Пользователь пишет тестовому аккаунту; WeChat вызывает webhook `wechat.php`, а PHP возвращает XML-ответ в рамках этого запроса. В текущем примере текст передаётся в ProxyAPI/OpenAI, затем ответ возвращается пользователю.
2. **Отправка из браузера.** Администратор входит в панель `wechat.php?console=1`, выбирает пользователя из списка подписчиков и отправляет текст через Customer Service Message API.

Панель и webhook находятся в одном `wechat.php`. Существующие настройки вебхука не меняются.

## Что понадобится

- тестовый аккаунт WeChat Official Accounts Platform;
- PHP-хостинг с HTTPS, PHP cURL и включёнными PHP-сессиями;
- для диалога с ИИ — API-ключ ProxyAPI/OpenAI;
- тестовый пользователь, который подписался на аккаунт через QR-код sandbox и написал ему хотя бы одно сообщение.

## Где получить AppID и AppSecret

Откройте [официальную страницу тестовой платформы WeChat](https://mp.weixin.qq.com/debug/cgi-bin/sandboxinfo?action=showinfo&t=sandbox/index) и войдите в WeChat. На странице sandbox отображаются AppID и AppSecret тестового аккаунта, QR-код для подписки, список пользователей и настройки интерфейса сообщений.

AppID/AppSecret нужны серверной панели для получения `access_token` и вызова API. Не вставляйте их в URL, браузерный JavaScript, публичный репозиторий или сообщения. Если секрет уже оказался публичным, перевыпустите его в панели WeChat.

## Установка и настройка

1. Загрузите `wechat.php` и `wechat_console_config.sample.php` на PHP-хостинг в каталог сайта.
2. Скопируйте пример в `wechat_console_config.php` и задайте там AppID, AppSecret и отдельный длинный пароль панели. Файл `wechat_console_config.php` исключён из Git. Если хостинг позволяет, держите его вне публичного каталога.
3. В панели хостинга задайте `WECHAT_TOKEN` — случайную строку из 3–32 английских букв и цифр. Также задайте `OPENAI_API_KEY`, если хотите включить ответ ИИ. Это переменные окружения PHP; файл `.env.example` приведён только как список имён и сам PHP его не загружает.
4. На официальной sandbox-странице укажите URL вида `https://ваш-домен/wechat.php` и тот же `WECHAT_TOKEN`, затем сохраните настройки интерфейса.
5. Откройте `https://ваш-домен/wechat.php?console=1`, войдите по паролю панели и обновите список пользователей.
6. Подпишитесь на тестовый аккаунт по его QR-коду и отправьте ему сообщение. Выберите пользователя в панели и выполните тестовую отправку.

Если вашему хостингу удобнее передавать AppID/AppSecret переменными окружения, используйте `WECHAT_APP_ID`, `WECHAT_APP_SECRET` и `WECHAT_CONSOLE_PASSWORD`. Локальный `wechat_console_config.php` имеет приоритет как резервный источник только когда соответствующая переменная окружения пуста.

## Как проходит обмен сообщениями

### Входящее сообщение и ответ пользователя через webhook

1. WeChat проверяет URL webhook GET-запросом с `signature`, `timestamp`, `nonce` и `echostr`. PHP проверяет подпись с `WECHAT_TOKEN` и возвращает исходный `echostr`.
2. Когда пользователь пишет тестовому аккаунту, WeChat отправляет на этот URL POST-запрос с XML, содержащим отправителя (`FromUserName`), получателя, тип и текст сообщения.
3. PHP проверяет подпись, читает XML и формирует XML-ответ с тем же пользователем в `ToUserName`.
4. Для текстового сообщения пример получает ответ модели через ProxyAPI и возвращает его WeChat в HTTP-ответе на webhook. Отдельный API вызов для такой немедленной реакции не нужен: это passive reply.

### Отправка сообщения из панели

1. Панель получает список подписчиков тестового аккаунта и показывает их никнеймы и сокращённые OpenID.
2. При отправке сервер PHP получает/переиспользует `access_token` и вызывает `POST /cgi-bin/message/custom/send` с `touser`, `msgtype: "text"` и `text.content`.
3. Панель показывает результат `errcode` WeChat API.

Customer Service Message API не является свободной личной перепиской: WeChat ограничивает отправку правилами и сроком после взаимодействия пользователя с аккаунтом. Сам факт подписки не гарантирует отправку; ориентируйтесь на код и текст ошибки API.

## Методы API, используемые панелью

| Метод | Назначение |
| --- | --- |
| `GET /cgi-bin/token` | Получить `access_token` по AppID и AppSecret. |
| `GET /cgi-bin/user/get` | Получить OpenID подписчиков тестового аккаунта. |
| `POST /cgi-bin/user/info/batchget` | Получить имена пользователей для списка. |
| `POST /cgi-bin/message/custom/send` | Отправить выбранному OpenID текстовое Customer Service сообщение. |

## Какие данные не публиковать

Не добавляйте в Git или публичные issues:

- `wechat_console_config.php`, AppSecret, `access_token`;
- `OPENAI_API_KEY`, учётные данные прокси;
- `wechat_log.txt` и логи хостинга — в них могут быть данные запросов и ошибок.

Панель требует HTTPS, отдельный пароль, PHP-сессию и CSRF-токен. Не оставляйте тестовую панель общедоступной без защиты на рабочем аккаунте.

## Официальная документация

- [WeChat Official Accounts Platform — тестовая учётная запись](https://mp.weixin.qq.com/debug/cgi-bin/sandboxinfo?action=showinfo&t=sandbox/index)
- [Официальная документация: получение access_token](https://developers.weixin.qq.com/doc/offiaccount/Basic_Information/Get_access_token.html)
- [Официальная документация: отправка Customer Service Message](https://developers.weixin.qq.com/doc/service/api/customer/message/api_sendcustommessage)
- [Официальная документация: список подписчиков](https://developers.weixin.qq.com/doc/offiaccount/User_Management/Getting_a_User_List.html)
- [Официальная документация: базовая информация пользователя](https://developers.weixin.qq.com/doc/offiaccount/User_Management/Get_users_basic_information_UnionID.html)
- [Официальная документация: настройка и разработка сервисного аккаунта](https://developers.weixin.qq.com/doc/service/guide/dev/)

Ссылки ведут на документы WeChat/Tencent; разделы и доступность отдельных методов могут различаться для sandbox и типа аккаунта.
