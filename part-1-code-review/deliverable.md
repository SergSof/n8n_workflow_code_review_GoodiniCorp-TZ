# Part 1 - Code Review - Отчёт

## Окружение

- Версия n8n, в которой импортировал: 1.115.3 Self Hosted
- ОС, на которой запускал: Ubuntu 24.04.3 LTS
- PostgreSQL для тестирования: PostgreSQL 16, отдельная БД `goodini_test_task` на host port `5433`
- Подключение из n8n к тестовой БД: `172.18.0.1:5433`
- Возникли ли проблемы при импорте: нет, workflow импортировался без ошибок

---

## 01-mini-register.json

### Краткое описание workflow

Workflow реализует мини-регистрацию пользователя.

Ожидаемый сценарий:

1. Webhook принимает `email`, `password`, `name`.
2. Code node валидирует входные данные.
3. PostgreSQL node проверяет, существует ли пользователь с таким email.
4. Если email уже существует — workflow возвращает ошибку.
5. Если email новый — workflow хеширует пароль, создаёт пользователя, сохраняет verification code.
6. HTTP Request node отправляет email через SendGrid.
7. Respond to Webhook возвращает успешный ответ клиенту.

---

### Найденные проблемы

#### Проблема 1: Некорректная логика проверки существующего email

- **Severity:** high
- **Где:** нода `Check Email Exists`, связь после неё
- **Что не так:**  
  Исходная нода делала запрос:

  ```sql
  SELECT id FROM users WHERE email = '{{ $json.email }}' LIMIT 1
  ```

  После этого не было IF-ноды, которая проверяет результат. Если email новый, SELECT возвращает 0 строк, и n8n останавливает workflow из-за отсутствия output data. Если email уже существует, workflow продолжает регистрацию дальше.

- **Чем грозит:**  
  Основной сценарий регистрации работает неправильно:
  - новый email может не пройти дальше;
  - уже существующий email может попасть в ветку создания пользователя;
  - возможны ошибки БД из-за unique constraint;
  - клиент не получает нормальный ответ `409 Email already registered`.

- **Как фиксить:**  
  Заменить SQL-запрос на запрос, который всегда возвращает одну строку:

  ```sql
  SELECT EXISTS (
    SELECT 1
    FROM users
    WHERE email = '{{ $json.email }}'
  ) AS email_exists;
  ```

  После этого добавить IF-ноду:
  - `true` → вернуть `409 Email already registered`;
  - `false` → продолжить регистрацию.

---

#### Проблема 2: `Insert User` берёт email из неправильного места

- **Severity:** high
- **Где:** нода `Insert User`, поле `query`
- **Что не так:**  
  В исходном query использовалось выражение:

  ```sql
  '{{ $json.body.email }}'
  ```

  Но после ноды `Hash Password` текущий `$json` уже не содержит `body.email`. В нём есть только `password_hash` и `code`.

- **Чем грозит:**  
  Пользователь может быть создан с пустым/неверным email, либо SQL-запрос упадёт. Это ломает основной сценарий регистрации.

- **Как фиксить:**  
  Брать email из ноды `Validate Input`, где он уже нормализован:

  ```sql
  INSERT INTO users (email, password_hash, name)
  VALUES (
    '{{ $("Validate Input").first().json.email }}',
    '{{ $json.password_hash }}',
    '{{ $("Validate Input").first().json.name }}'
  )
  RETURNING id;
  ```

---

#### Проблема 3: SendGrid Bearer token захардкожен в workflow

- **Severity:** high
- **Где:** нода `Send Email`, header `Authorization`
- **Что не так:**  
  В HTTP Request node напрямую указан SendGrid Bearer token. При экспорте workflow этот токен попадает в JSON.

- **Чем грозит:**  
  При публикации workflow в GitHub или передаче архива токен будет скомпрометирован. Через него можно отправлять письма от имени аккаунта и тратить лимиты SendGrid.

- **Как фиксить:**  
  Bearer token нужно вынести из workflow. Возможны два варианта:

  1. Использовать environment variable, например `SENDGRID_API_KEY`, и в header указать:

     ```js
     ={{ 'Bearer ' + $env.SENDGRID_API_KEY }}
     ```

     Реальное значение ключа хранится на сервере / в docker-compose / `.env`, но не попадает в exported workflow.

  2. Использовать n8n Credentials для HTTP Request node, чтобы токен хранился в credentials, а не в exported JSON.

  В рамках тестового задания реальный SendGrid API не вызывался. Проверялась структура workflow и корректность передачи email/code в тело запроса.

---

#### Проблема 4: SQL-запросы собираются через строковую подстановку

- **Severity:** high / medium
- **Где:** `Check Email Exists`, `Insert User`, `Save Code`
- **Что не так:**  
  Пользовательские данные вставляются напрямую в SQL-строку через expressions.

  Пример:

  ```sql
  WHERE email = '{{ $json.email }}'
  ```

- **Чем грозит:**  
  Возможны SQL injection, ошибки на кавычках и некорректная обработка спецсимволов.

- **Как фиксить:**  
  Использовать параметризованные запросы PostgreSQL node, если это доступно в текущей версии n8n. Минимум — строго валидировать поля и не вставлять пользовательский ввод напрямую в SQL.

---

#### Проблема 5: `name` валидируется, но не сохраняется

- **Severity:** medium
- **Где:** `Validate Input`, `Insert User`
- **Что не так:**  
  Workflow требует поле `name`, но в исходной версии при создании пользователя сохранялись только `email` и `password_hash`.

- **Чем грозит:**  
  Клиент передаёт обязательное поле, но workflow его теряет. Это создаёт неочевидное поведение и может ломать дальнейшую бизнес-логику, если имя ожидается в профиле пользователя.

- **Как фиксить:**  
  Либо сохранять `name` в таблицу `users`, либо убрать его из обязательных входных полей. В fixed workflow имя сохраняется в `users.name`.

---

#### Проблема 6: Нет нормальной обработки ошибок и HTTP status codes

- **Severity:** medium
- **Где:** `Validate Input`, общий error flow
- **Что не так:**  
  При ошибке валидации Code node делает `throw new Error(...)`. Отдельной ветки для ответа клиенту нет.

- **Чем грозит:**  
  Клиент может получить технический 500 или timeout вместо понятного ответа:

  ```json
  {
    "success": false,
    "error": "Invalid email"
  }
  ```

- **Как фиксить:**  
  Для пользовательских ошибок не бросать исключение напрямую, а возвращать структурированный результат и обрабатывать его через IF-ноды. Например:
  - invalid email → 400;
  - password too short → 400;
  - email already exists → 409.

---

#### Проблема 7: Нет транзакционности между созданием пользователя и verification code

- **Severity:** medium
- **Где:** `Insert User`, `Save Code`, `Send Email`
- **Что не так:**  
  Пользователь создаётся в одной ноде, verification code сохраняется в другой, email отправляется в третьей. Эти действия не объединены в транзакцию.

- **Чем грозит:**  
  Возможны частичные состояния:
  - user создан, но code не сохранился;
  - user создан, code сохранился, но email не отправился;
  - клиент получил ошибку, но запись в БД уже появилась.

- **Как фиксить:**  
  Минимально — добавить статусы пользователя и возможность повторной отправки кода. Лучше — объединить создание пользователя и verification code в один SQL-запрос/транзакцию.

---

#### Проблема 8: Verification code генерируется через `Math.random`

- **Severity:** medium
- **Где:** нода `Hash Password`
- **Что не так:**  
  Код создаётся через `Math.random()`, который не предназначен для security-sensitive сценариев.

- **Чем грозит:**  
  Код подтверждения менее надёжен, чем мог бы быть.

- **Как фиксить:**  
  Использовать crypto-safe генерацию:

  ```js
  const crypto = require('crypto');
  const code = String(crypto.randomInt(100000, 1000000));
  ```

---

#### Проблема 9: Verification code хранится в открытом виде

- **Severity:** low / medium
- **Где:** нода `Save Code`
- **Что не так:**  
  Код подтверждения сохраняется в БД как plain text.

- **Чем грозит:**  
  При доступе к БД активные коды можно использовать напрямую.

- **Как фиксить:**  
  Хранить hash кода и при проверке сравнивать hash, а не исходное значение.

---

#### Проблема 10: Внешний модуль `bcryptjs` недоступен в текущем n8n окружении

- **Severity:** medium
- **Где:** нода `Hash Password`
- **Что не так:**  
  Нода использует `require('bcryptjs')`, но в текущем self-hosted Docker-окружении n8n этот пакет не установлен. При выполнении нода падает с ошибкой `Cannot find module 'bcryptjs'`.

- **Чем грозит:**  
  Workflow успешно импортируется, но падает на runtime при регистрации пользователя.

- **Как фиксить:**  
  Один из вариантов:
  - собрать кастомный Docker image n8n с установленным `bcryptjs`;
  - разрешить внешний модуль через `NODE_FUNCTION_ALLOW_EXTERNAL=bcryptjs`;
  - заменить реализацию на встроенный Node.js `crypto`;
  - вынести hashing в отдельный backend-сервис.

---

## 02-job-poller.json

### Найденные проблемы

Пока не разобран.

---

## 03-file-uploader.json

### Найденные проблемы

Пока не разобран.

---

## Топ-3 для фикса

1. **[WF 01] Некорректная проверка существующего email**  
   Это ломает основной сценарий регистрации. Без IF-ноды workflow не различает новый и уже существующий email.

2. **[WF 01] `Insert User` берёт email из неправильного места**  
   После `Hash Password` в текущем `$json` уже нет `body.email`, поэтому создание пользователя работает некорректно.

3. **[WF 01] Hardcoded SendGrid Bearer token**  
   Это критичная security-проблема. Секреты нельзя хранить внутри JSON workflow.

---

## Реализованные фиксы

В папке `workflows/fixed/` лежит:

- `01-mini-register.json`

Изменения:

1. В `Check Email Exists` SQL заменён на `SELECT EXISTS (...) AS email_exists`, чтобы нода всегда возвращала одну строку.
2. Добавлена IF-нода `IF Email Exists`.
3. Добавлена ветка ответа `409 Email already registered`.
4. В `Insert User` email теперь берётся из `Validate Input`, а не из несуществующего `$json.body.email`.
5. В `Insert User` добавлено сохранение `name`, так как поле валидируется как обязательное.
6. В `Send Email` удалён hardcoded SendGrid Bearer token. Вместо него используется environment variable `SENDGRID_API_KEY`.
7. `Send Email` оставлен в успешной ветке после `Save Code`, а duplicate-email ветка не отправляет письмо.

---

## Ручная проверка `01-mini-register.json`

Workflow был импортирован в self-hosted n8n и проверен через test webhook.

### Проверка нового пользователя

Запрос:

```bash
curl -i -X POST "https://guthatosisoy.beget.app/webhook-test/mini-register" \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"123456","name":"Sergey"}'
```

Результат:

- `Validate Input` вернул нормализованные поля `email`, `password`, `name`.
- `Check Email Exists` вернул `email_exists: false`.
- `Insert User` создал пользователя и вернул UUID.
- `Save Code` сохранил verification code.
- `Respond OK` вернул успешный ответ.

Проверка БД:

```sql
SELECT id, email, name, created_at FROM users;
```

Результат: пользователь `test@example.com` создан.

```sql
SELECT user_id, code, expires_at, created_at FROM verification_codes;
```

Результат: verification code создан и связан с пользователем.

### Проверка повторного email

Повторный запрос с тем же email ушёл в ветку `Respond Email Exists`.

Ответ:

```json
{
  "success": false,
  "error": "Email already registered"
}
```

Это подтверждает, что исправленная проверка уникальности email работает корректно.

### Проверка SendGrid

Реальный SendGrid API не вызывался, так как в задании указано, что реальный вызов SendGrid не требуется. Была проверена структура ноды `Send Email`:

- письмо отправляется только после создания пользователя и сохранения verification code;
- duplicate-email ветка не отправляет письмо;
- email получателя берётся из `Validate Input`;
- verification code берётся из `Hash Password`;
- Bearer token вынесен в `SENDGRID_API_KEY`.

---

## Не сделано, но стоит

- Переписать SQL-запросы на параметризованные.
- Добавить полноценную обработку ошибок с корректными HTTP status codes.
- Добавить транзакционность при создании пользователя и verification code.
- Добавить повторную отправку verification code.
- Хранить verification code в виде hash.
- Заменить `Math.random()` на crypto-safe генерацию.
- Заменить временный тестовый hash на production-ready hashing: bcrypt/argon2 через контролируемое окружение или отдельный backend-сервис.
- Подключить реальный SendGrid credential и verified sender для end-to-end проверки отправки email.

---

## Общее впечатление от workflow

Workflow хорошо показывает базовую идею регистрации через webhook: принять данные, проверить email, создать пользователя, сохранить код и отправить письмо.

Основные проблемы не в самой идее, а в деталях реализации: неверная передача данных между нодами, отсутствие ветвления после проверки email, hardcoded secret, слабая обработка ошибок и SQL через строковые expressions. Если бы я проектировал этот workflow с нуля, я бы сначала описал состояния регистрации, HTTP-коды, схему таблиц и транзакционность, а потом уже собирал n8n-логику вокруг этих правил.

При этом исправления в fixed workflow не являются переписыванием архитектуры. В исходном workflow уже была отдельная нода `Check Email Exists`, но её результат не использовался для ветвления. Добавленная IF-нода только реализует ожидаемую бизнес-логику: новый email продолжает регистрацию, существующий email получает ответ `409 Email already registered`.
