# Bakery Production Planning API (Lab 5: JWT Access/Refresh)

REST API для управления производством пекарни с JWT-аутентификацией (access + refresh) и серверным учётом сессий.

## Стек

- Java 21, Spring Boot, Maven
- Spring Web MVC, Spring Data JPA, Hibernate
- PostgreSQL
- Spring Security + JWT (jjwt, HS256)
- dotenv-java

## JWT-аутентификация

### Токены

- Access-токен: срок жизни 15 минут, передаётся в заголовке Authorization: Bearer <token>.
- Refresh-токен: срок жизни 7 дней, передаётся в JSON-теле при обновлении.
- Подпись: HS256.
- Каждый токен содержит уникальный jti (UUID), sub (email), type (access/refresh), iat, exp.
- Access-фильтр принимает только токены с type=access. Обработчик обновления — только с type=refresh.

### Эндпоинты

POST /auth/login — проверка логина/пароля, выдача пары. Успех: 200 + пара токенов.
POST /auth/register — создание учётной записи + выдача пары. Успех: 201 + пара токенов.
POST /auth/refresh — обменять refresh на новую пару. Успех: 200 + новая пара.

### Серверный учёт сессий

Каждая выданная пара сохраняется в таблице user_sessions:

- id (UUID), username, device_id (nullable), access_token (TEXT), refresh_token (TEXT, unique), access_token_expiry, refresh_token_expiry, status (ACTIVE / USED / REVOKED).
- Одна запись = одна пара токенов.
- При обновлении старая запись переходит в USED, для новой пары создаётся новая строка ACTIVE.
- При повторном использовании старого refresh — старая запись переходит в REVOKED и возвращается 401.
- Все операции выполняются в одной транзакции (@Transactional).

### Коды ошибок

400 — отсутствующее/пустое обязательное поле.
401 — неверные учётные данные, невалидный/просроченный refresh, неизвестная/использованная/отозванная сессия, неверный тип токена.
409 — повторная регистрация существующего email.
Ответ с ошибкой не содержит новую пару токенов.

### Роли

USER — чтение, создание своих заявок, удаление только своих заявок.
ADMIN — полный доступ + бизнес-операции + управление пользователями.

## Предметная область

1. User — пользователь (заказчик).
2. Ingredient — ингредиент.
3. Recipe — рецептура.
4. ProductionBatch — партия.
5. BakingRequest — заявка на выпечку.
6. UserSession — серверная сессия (пара токенов).

## Бизнес-операции

1. POST /api/business/start-batch/{id} — запуск партии со списанием ингредиентов.
2. POST /api/business/complete-batch/{id} — завершение партии.
3. POST /api/business/request-to-batch/{id} — превращение заявки в партию.
4. POST /api/business/plan-batch — массовое планирование.
5. POST /api/business/close-batch/{id} — закрытие партии с обновлением заявок.

## Запуск

### Требования

- Java 21, Maven, PostgreSQL 14+
- Файл .env в корне (см. .env.example)

### Настройка БД

CREATE DATABASE bakery;
CREATE USER bakery_user WITH PASSWORD 'bakery_pass';
GRANT CONNECT ON DATABASE bakery TO bakery_user;
GRANT USAGE ON SCHEMA public TO bakery_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO bakery_user;
GRANT CREATE ON SCHEMA public TO bakery_user;

### .env

DB_URL=jdbc:postgresql://localhost:5432/bakery
DB_USERNAME=bakery_user
DB_PASSWORD=bakery_pass
JWT_SECRET=<секретный ключ минимум 32 символа>

### Запуск

./mvnw spring-boot:run

Сервер: http://localhost:8080

### Тестовые учётные записи

- admin@bakery.local / admin123 (ADMIN)
- user@bakery.local / user123 (USER)

## Тестирование

Файл requests.http содержит полный сценарий:

1. Регистрация Charlie → 201 + пара.
2. Логин admin → 200 + пара.
3. Логин user → 200 + пара.
4. Неверный пароль → 401.
5. Повторная регистрация → 409.
6. Слабый пароль → 400.
7. GET /api/users под admin → 200.
8. GET /api/users под user → 403.
9. POST /api/ingredients под admin → 201.
10. POST /api/ingredients под user → 403.
11. POST /auth/refresh → 200 + новая пара.
12. Повторный refresh старого токена → 401, сессия → REVOKED.
