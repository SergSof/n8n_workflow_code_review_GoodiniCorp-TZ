# n8n workflow code review — GoodiniCorp TZ

Repository for the GoodiniCorp n8n test task.

## Scope

The task is focused on reviewing existing n8n workflows, finding issues, ranking them by severity, and applying targeted fixes without redesigning the whole architecture.

## Current status

### Part 1 — Code Review

Reviewed workflow:

- `01-mini-register.json`

Implemented fixes for the top issues:

1. Fixed email existence check so the workflow does not stop on a new email.
2. Added explicit branching for an existing email with a `409` response.
3. Fixed user insert expression: email is now taken from `Validate Input`.
4. Moved SendGrid Bearer token out of the workflow JSON by using `SENDGRID_API_KEY`.

Manual checks were performed in self-hosted n8n:

- successful registration path: user is inserted into PostgreSQL and verification code is saved;
- duplicate email path: workflow returns `Email already registered` and does not send email.

## Repository structure

```text
part-1-code-review/
  deliverable.md
  workflows/
    original/
      01-mini-register.json
    fixed/
      01-mini-register.json
TELEGRAM.md
```

## Notes

The original workflow contains a hardcoded SendGrid token issue. In the public repository, the token value in the original workflow is redacted intentionally to avoid publishing a secret-looking value.

The real SendGrid API was not called, because the task description says the external SendGrid call does not need to be executed for this review.

## Contact

See `TELEGRAM.md`.
