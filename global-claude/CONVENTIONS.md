# CONVENTIONS.md

## Commit messages
Follow Conventional Commits. Format: {type}({scope}): {description}

Types: feat, fix, test, refactor, perf, docs, chore, db, ci
Scope: the service or module name (order, payment, inventory, k8s, deps)

Examples:
feat(person): add new person use case
fix(person): handle invalid data in person use case
test(person): add integration test for CreatePersonUseCase
db(person): add phone_number migration 2026-001
chore(deps): bump spring-boot to 3.4.1

## Branch names
Pattern: {type}/{short-description}

Examples:
feat/person-new-use-case
fix/person-invalid-data-in-use-case
db/person-phone-column

## Logging
- Never log passwords, tokens, card numbers or emails
- Log IDs and correlation values only
- Use structured JSON with traceId in every log line

## PR rules
- Max 400 lines changed per PR
- Every PR needs a Liquibase migration if schema changes
