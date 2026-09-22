---
description: "Never run database migrations"
trigger: always_on
---

# Never Run Migrations

- **Never run database migration commands**, in any repository:
  `python manage.py migrate`, `python manage.py makemigrations`,
  or any equivalent (e.g. `sqlmigrate`, ORM/schema migration tools).
- Do not run them even if a task seems to require it — e.g. tests
  failing due to missing tables or columns.
- If a migration appears necessary, stop and tell the user what
  command you believe is needed and why, and let them run it.
