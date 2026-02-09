# Paperless-ngx

[Paperless-ngx](https://github.com/paperless-ngx/paperless-ngx) is a community-supported supercharged document management system: scan, index and archive all your documents.

## .env

Configure as required and ensure you set a `PAPERLESS_SECRET_KEY` ([docs](https://docs.paperless-ngx.com/configuration/?h=secret#PAPERLESS_SECRET_KEY)):

```
USERMAP_UID=1000
USERMAP_GID=1000
PAPERLESS_REDIS=redis://paperless-broker:6379
PAPERLESS_URL=https://paperless.example.com
PAPERLESS_SECRET_KEY=
PAPERLESS_TIME_ZONE=Europe/London
PAPERLESS_APP_TITLE=Paperless
PAPERLESS_EMAIL_TASK_CRON=*/15 * * * *
PAPERLESS_TRAIN_TASK_CRON=30 */3 * * *
PAPERLESS_INDEX_TASK_CRON=40 0 * * *
PAPERLESS_SANITY_TASK_CRON=00 13 * * tue
```