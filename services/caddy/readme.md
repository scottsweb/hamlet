# Caddy

[Caddy](https://caddyserver.com/) is a powerful, extensible platform to serve your sites, services, and apps, written in Go.

For this server, Caddy acts as a reverse proxy to the containers. It also configures SSL automatically for each sub-domain.

## Building

Caddy is built manually using `caddy-update.service` to enable the following plugins:

* [caddy-docker-proxy](https://github.com/lucaslorentz/caddy-docker-proxy) — to allow configuration via container labels
* [sablier](https://sablierapp.dev/) — to scale some containers to zero
* [caddy-ratelimit](https://github.com/mholt/caddy-ratelimit)
* [caddy-cloudflare-ip](https://github.com/WeidiDeng/caddy-cloudflare-ip)

Place `caddy-update.service` and `caddy-update.timer` in `~/.config/systemd/user/`. Eventually I will switch this to use a `.build` Quadlet ([more info](https://github.com/containers/podman/pull/22694)).

## .env

Tweak the settings to add your Cloudflare token and email. 

```dotenv
CLOUDFLARE_API_TOKEN=
CADDY_INGRESS_NETWORKS=proxy
EMAIL=
CADDY_DOCKER_SCAN_STOPPED_CONTAINERS=true
CADDY_DOCKER_POLLING_INTERVAL=5s
CADDY_DOCKER_NO_SCOPE=true
```

## Firewall

```bash
sudo firewall-cmd --zone=FedoraServer --add-service=http
sudo firewall-cmd --zone=FedoraServer --add-service=http --permanent
sudo firewall-cmd --zone=FedoraServer --add-service=https
sudo firewall-cmd --zone=FedoraServer --add-service=https --permanent
```

`firewall-cmd --get-default-zone` will let you know which zone you are currently using. Use `sudo firewall-cmd --reload` to reload the firewall.

## Fixing buffer warnings (failed to sufficiently increase receive buffer size)

You might see [buffer warnings in your logs](https://github.com/quic-go/quic-go/wiki/UDP-Buffer-Sizes) and this can be fixed using the [following advice](https://github.com/quic-go/quic-go/issues/3801#issuecomment-2985154520).

Edit the following file `sudo nano /etc/sysctl.d/99-buffer-size.conf` and add the following:

```bash
net.core.rmem_max=7500000
net.core.wmem_max=7500000
```

Reload your config changes: `sudo sysctl -p /etc/sysctl.d/99-buffer-size.conf` and check they have been correctly set: `sudo sysctl net.core.rmem_max` and `sudo sysctl net.core.wmem_max`.