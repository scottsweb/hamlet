# Miniflux

[Miniflux](https://miniflux.app/) is a minimalist and opinionated feed reader.

## .env

Replace `!password` with your chosen database password.

```bash
DATABASE_URL=postgres://miniflux:!password@miniflux-db/miniflux?sslmode=disable
RUN_MIGRATIONS=1
POSTGRES_USER=miniflux
POSTGRES_PASSWORD=!password
POLLING_FREQUENCY=240
BATCH_SIZE=10
```