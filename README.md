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
