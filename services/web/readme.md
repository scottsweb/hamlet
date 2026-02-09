# Web

This is a two container pod running [Static Web Server](https://static-web-server.net/) and [Cloudflared](https://github.com/cloudflare/cloudflared) to server a static website. Static Web Server serves the files and Cloudflared is used to tunnel from my home network to Cloudflare.

## .env

Set your Cloudflare token and customise as necessary:

```dotenv
SERVER_PORT=80
SERVER_ROOT=/public
SERVER_ERROR_PAGE_404=/public/404.html
CLOUDFLARE_WEB_TOKEN=!secret
```