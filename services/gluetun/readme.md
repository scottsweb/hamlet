# Gluetun

[Gluetun](https://github.com/qdm12/gluetun) is VPN client in a thin Docker container for multiple VPN providers, written in Go, and using OpenVPN or Wireguard, DNS over TLS, with a few proxy servers built-in. 

Gluetun [needs to be run as root by default](https://github.com/qdm12/gluetun/issues/1519) (although there are some tips in that link on how to run it rootless). This suits my needs for now as I mainly use it with some of the apps in Wolf, which is also running as root.

Place the `.container` file in `/etc/containers/systemd/gluetun.container`.

## .env

Configure as required.

```bash
VPN_SERVICE_PROVIDER=
VPN_TYPE=wireguard
WIREGUARD_PRIVATE_KEY=
SERVER_COUNTRIES=
STREAM_ONLY=on
```