# Containership — каталог контейнеровозов

[Бэкенд](https://github.com/oravailable/loading_backend) · [Фронтенд](https://github.com/oravailable/loading_frontend)

## Технологии

Go, Gin, GORM, PostgreSQL 16, MinIO, Redis 7, JWT и Swagger (OpenAPI).

Документ описывает текущую локальную реализацию проекта; загрузка исходного кода в этот репозиторий выполняется отдельно.

Сервис на Go и Gin для просмотра и публикации контейнеровозов. Вместимость указана в TEU, осадка — в метрах.

## Добавление

Перед «Далее» передаются **три поля**: название name, фото image и видео video. В API файлы обязательны. В HTML-форме можно оставить медиа по умолчанию.

В черновике вводятся четыре предметных поля: тип ship_type, описание short_description, вместимость capacity и осадка draft. Описание, вместимость и осадка обязательны для публикации, тип можно опустить. Эти четыре колонки допускают NULL до заполнения. formed_at — отдельное системное время публикации.

id, status, creator_id, formed_at и пути медиа определяет сервер. Файлы сохраняются в MinIO. Один пользователь имеет не более одного черновика. Удаление логическое, повторный лайк не увеличивает счётчик.

## HTTP методы

| № | Метод | URL | Вход | Выход |
|---|---|---|---|---|
| 01 Регистрация | POST | /api/users/register | JSON: password string; login?, name? string; также text/plain с паролем | 201; data.id integer, data.login string, data.name string |
| 02 Аутентификация | POST | /api/users/auth | JSON: login string, password string | 200; data.access_token string, data.token_type string, data.expires_in integer, data.expires_at string, data.user object |
| 03 Список с фильтром | GET | /api/containerships | Query minCapacity?, maxCapacity? integer, TEU | 200; data array карточек, is_creator integer 0/1 |
| 04 Далее | POST | /api/containerships | multipart/form-data: name string, image file, video file | 201; data object карточки в статусе draft |
| 05 Черновик | GET | /api/containerships/draft | Без ID и тела. JWT | 200; data object карточки или null |
| 06 Опубликовать | PUT | /api/containerships/publish | JSON: short_description string, capacity integer, draft number, ship_type? string | 200; data object, status published, обновлённые поля |
| 07 Лента | GET | /api/containerships/feed | Query id? integer, next? boolean, minCapacity?, maxCapacity? integer | 200; data.containership object/null, is_liked integer 0/1, like_count integer, has_next boolean |
| 08 Лайк | POST | /api/containerships/{id}/like | Path id integer. JSON liked integer 0/1 | 200; data.liked integer 0/1, data.like_count integer |
| 09 Удаление | DELETE | /api/containerships/{id} | Path id integer; своя опубликованная карточка | 200; data.id integer, data.deleted boolean |
| 10 Деавторизация | POST | /api/users/logout | Без тела. JWT | 200; data.logged_out boolean |

Лента без id возвращает первую карточку. Для следующей используйте id и next=true. После последней выдаётся первая подходящая. Служебный GET /api/media/{name} возвращает двоичный файл и не входит в десять методов бизнес-сценария. Ошибка содержит error string.

## Доступ

Вход выполняется в Swagger через POST /api/users/auth. Полученный data.access_token вставляется в Authorize, а в Postman — в явный заголовок Authorization: Bearer <JWT>. Публичный список работает без JWT. Публикация гостем получает 401. Владелец публикует свой черновик. POST /api/users/logout записывает jti в Redis blacklist до истечения токена. После выхода старый токен получает 401.

## Таблицы PostgreSQL

### users

| Колонка | Тип | Назначение |
|---|---|---|
| id | BIGINT PK | идентификатор пользователя |
| name | VARCHAR(100) | имя |
| login | VARCHAR(50) UNIQUE | логин |
| password_hash | VARCHAR(100) | bcrypt-хэш, скрыт из JSON |

### containerships

| Колонка | Тип | Назначение |
|---|---|---|
| id | BIGINT PK | идентификатор карточки |
| name | VARCHAR(150) | название |
| image_url | TEXT | путь к фотографии |
| video_url | TEXT | путь к видео |
| ship_type | VARCHAR(80) NULL | тип; необязателен для черновика |
| short_description | VARCHAR(500) NULL | описание; заполняется при публикации |
| capacity | BIGINT NULL | вместимость, TEU; заполняется при публикации |
| draft | NUMERIC NULL | осадка, м; заполняется при публикации |
| status | VARCHAR(20) | draft → published → deleted |
| creator_id | BIGINT FK users.id | назначает сервер |
| formed_at | TIMESTAMPTZ NULL | системное время публикации, вычисляет сервер |

### containership_likes

| Колонка | Тип | Назначение |
|---|---|---|
| id | BIGINT PK | идентификатор отметки |
| user_id | BIGINT FK users.id | автор отметки |
| containership_id | BIGINT FK containerships.id | карточка. UNIQUE(user_id, containership_id) |

## Запуск

PostgreSQL 16, MinIO и Redis 7 описаны в docker-compose.yml. Запустите существующие контейнеры, задайте DATABASE_DSN и выполните go run ./cmd/containership -addr :8094. Если контейнеры уже созданы, используйте docker start с их именами. Миграции находятся в migrations. Перед изменением существующей БД сделайте резервную копию.

## Форматы данных и ограничения

Ответ успешного запроса обёрнут в `data`, ошибка имеет вид `{"error":"описание"}`.
Карточка содержит `id: integer`, `name: string`, `ship_type: string`, `short_description: string`,
`capacity: integer`, `draft: number`, `status: string`, `image_url: string`, `video_url: string`,
`creator_id: integer`, `formed_at: string(date-time) | null`, `like_count: integer`.
Список возвращает `data: array` и добавляет каждой карточке `is_creator: integer (0/1)`.
Лента возвращает `data: {containership: object | null, is_liked: integer (0/1), like_count: integer, has_next: boolean}`.
GET черновика возвращает карточку либо `data: null`.

Перед «Далее» разрешены только текстовое поле `name` и два файловых поля `image`, `video`.
Передача `capacity`, `draft`, `ship_type`, `short_description` или системных полей на первом шаге возвращает 400.
API требует файлы; браузерная форма может использовать медиа по умолчанию. URL файлов с клиента не принимаются.
В PostgreSQL четыре предметных поля нового черновика равны NULL; в JSON модели пустые строки и нули означают, что значения ещё не заполнены.

Публикация принимает JSON:

```json
{"ship_type":"Post-Panamax","short_description":"Контейнеровоз для морских перевозок.","capacity":18000,"draft":14.5}
```

`ship_type` можно опустить. `short_description` обязателен, `capacity` — целое от 1 до 100000,
`draft` — конечное число от 0.01 до 30 метров. Ограничения проверяет сервер.
`id`, `creator_id`, `status`, `formed_at` и имена медиа назначает сервер.
Отмена публикации в черновик не предусмотрена; удаление меняет статус published на deleted.
Повторный `liked=1` не добавляет вторую отметку; `liked=0` снимает её.
В публичный список и ленту попадают только published. Лента циклическая в пределах фильтра.

Основные ошибки: 400 — входные данные, 401 — нет действительного JWT, 404 — нет своей записи,
409 — уже есть черновик или занят логин, 503 — недоступны обязательные хранилища.
Служебный GET `/api/media/{name}` не входит в десять бизнес-методов.


## Проверка API

В локальном запуске на порту 8094 Swagger доступен по адресу http://127.0.0.1:8094/swagger/index.html.
Для демонстрации входа откройте его в режиме инкогнито: выполните регистрацию, затем `POST /api/users/auth`, скопируйте `data.access_token` и вставьте в **Authorize**.
В Postman используйте заголовок `Authorization: Bearer <JWT>`.

В отдельном варианте без авторизации создатель зафиксирован сервером через `currentCreator()` и `fixedCreatorID = 1`; методы входа и выхода являются заглушками. Для него используется отдельный запуск на порту 8093 и отдельная база данных. Обе версии имеют одинаковый набор из десяти бизнес-методов.

## Примеры входных данных

Регистрация — `POST /api/users/register`, `Content-Type: application/json`:

```json
{"login":"containership_reader","password":"your-password","name":"Александр"}
```

Аутентификация — `POST /api/users/auth`:

```json
{"login":"containership_reader","password":"your-password"}
```

Создание черновика — `POST /api/containerships`, `multipart/form-data`:

| Поле | Тип | Значение |
|---|---|---|
| name | Text | Название контейнеровоза |
| image | File | Фотография |
| video | File | Короткое видео |

При отправке файлов заголовок Content-Type с boundary формирует клиент. После «Далее» четыре предметных поля заполняются отдельным PUT публикации, приведённым выше.

Лайк — `POST /api/containerships/{id}/like`:

```json
{"liked":1}
```

Для отмены лайка передайте `{"liked":0}`.

## Связи таблиц

`containerships.creator_id → users.id`: один пользователь создаёт несколько карточек.
`containership_likes.user_id → users.id` и `containership_likes.containership_id → containerships.id`: связь пользователей и карточек через отметки.
Уникальность пары `(user_id, containership_id)` исключает повторный лайк одной карточки одним пользователем.
