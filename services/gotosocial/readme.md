# GoToSocial

[GoToSocial](https://gotosocial.org/) is an ActivityPub social network server, written in Golang. GoToSocial provides a lightweight, customizable, and safety-focused entryway into the [Fediverse](https://en.wikipedia.org/wiki/Fediverse).

You probably won't want to copy this setup directly as we run two separate instances of GTS on our server. This `.pod` is exposed to the Internet via a [Cloudflare tunnel](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/) and is not exposed to the local network.

## .env

Set your Cloudflare token for the `gts-cloudflare.container` container in your `.env` file.

```
CLOUDFLARE_FEDIVERSE_TOKEN=
```

## .instance.env

Example settings for your `gts.container`(s). A full list of settings can be found in the [GoToSocial docs](https://docs.gotosocial.org/en/latest/configuration/#environment-variables).

```dotenv
GTS_HOST=toot.example.com
GTS_PORT=8086
GTS_APPLICATION_NAME=GTS
GTS_ACCOUNTS_ALLOW_CUSTOM_CSS=true

GTS_ACCOUNTS_REGISTRATION_OPEN=false
GTS_ACCOUNTS_REASON_REQUIRED=false
GTS_MEDIA_IMAGE_MAX_SIZE=10485760
GTS_MEDIA_CLEANUP_FROM=14:30
GTS_MEDIA_CLEANUP_EVERY=72h
GTS_STATUSES_MAX_CHARS=1000

GTS_DB_TYPE=sqlite
GTS_DB_ADDRESS=/gotosocial/storage/sqlite.db
GTS_LETSENCRYPT_ENABLED=false
```