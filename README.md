# Code review n8n workflow — GoodiniCorp TZ

Репозиторий с выполнением тестового задания GoodiniCorp для позиции n8n-инженера.

## Цель задания

Задача — разобрать существующие n8n workflow, найти ошибки, расставить их по приоритету и внести точечные исправления без переписывания всей архитектуры.

Основной фокус:

- чтение чужого workflow;
- поиск логических, security и runtime-проблем;
- понимание передачи данных между n8n-нодами;
- аккуратные минимальные фиксы;
- оформление результата в виде markdown-отчёта и исправленных JSON-файлов.

## Текущее состояние

### Part 1 — Code Review

Разобран workflow:

- `01-mini-register.json`

Workflow реализует мини-регистрацию пользователя:

```text
Webhook
→ Validate Input
→ Check Email Exists
→ IF Email Exists
    true  → Respond Email Exists
    false → Hash Password
             → Insert User
             → Save Code
             → Send Email
             → Respond OK
````

## Что исправлено

В fixed-версии workflow внесены следующие изменения:

1. Исправлена проверка существующего email.

   * Исходный SQL мог вернуть 0 строк, из-за чего n8n останавливал выполнение.
   * Теперь используется `SELECT EXISTS (...) AS email_exists`, который всегда возвращает одну строку.

2. Добавлено явное ветвление через `IF Email Exists`.

   * Если email уже существует — workflow возвращает ошибку.
   * Если email новый — workflow продолжает регистрацию.

3. Добавлен ответ для существующего email.

   * Duplicate email больше не идёт в ветку создания пользователя.
   * Workflow возвращает:

   ```json
   {
     "success": false,
     "error": "Email already registered"
   }
   ```

4. Исправлен `Insert User`.

   * В исходной версии email брался из несуществующего `$json.body.email` после ноды `Hash Password`.
   * В fixed-версии email берётся из `Validate Input`.

5. Добавлено сохранение `name`.

   * Поле `name` валидируется как обязательное, поэтому оно также сохраняется в таблицу `users`.

6. Убран hardcoded SendGrid Bearer token.

   * В исходном workflow токен был прописан прямо в HTTP Request node.
   * В fixed-версии используется environment variable:

   ```js
   ={{ 'Bearer ' + $env.SENDGRID_API_KEY }}
   ```

7. `Send Email` оставлен в успешной ветке.

   * Письмо отправляется только после создания пользователя и сохранения verification code.
   * Duplicate-email ветка не отправляет письмо.

## Ручная проверка

Workflow был импортирован и проверен в self-hosted n8n.

Окружение:

* n8n: `1.115.3 Self Hosted`
* OS: `Ubuntu 24.04.3 LTS`
* PostgreSQL: отдельная тестовая БД `goodini_test_task`
* Подключение из n8n к БД: `172.18.0.1:5433`

Проверены сценарии:

* успешная регистрация нового пользователя;
* запись пользователя в таблицу `users`;
* запись verification code в таблицу `verification_codes`;
* повторная регистрация с тем же email;
* корректный ответ `Email already registered` для duplicate email.

Реальный SendGrid API не вызывался, так как по условиям задания внешний вызов SendGrid проверять не требуется. Проверялась структура ноды и корректность передачи email/code.

## Структура репозитория

```text
part-1-code-review/
  deliverable.md
  workflows/
    original/
      01-mini-register.json
    fixed/
      01-mini-register.json
TELEGRAM.md
README.md
```

## Файлы

* `part-1-code-review/deliverable.md` — отчёт по найденным проблемам, severity и реализованным фиксам.
* `part-1-code-review/workflows/original/01-mini-register.json` — исходный workflow.
* `part-1-code-review/workflows/fixed/01-mini-register.json` — исправленная версия workflow.
* `TELEGRAM.md` — контакт в Telegram.

## Важные замечания

* В публичной версии исходного workflow SendGrid token отредактирован / скрыт, чтобы не публиковать секретоподобное значение в GitHub.
* В `Hash Password` для ручной проверки использовался тестовый hash, потому что в текущем self-hosted n8n окружении внешний модуль `bcryptjs` недоступен.
* Для production нужно заменить тестовый hash на полноценный hashing: bcrypt/argon2 через контролируемое окружение, кастомный n8n image или отдельный backend-сервис.
* SQL-запросы стоит дополнительно переписать на параметризованные, если это поддерживается используемой PostgreSQL-ноды.

## Контакт

См. `TELEGRAM.md`.

```
```
