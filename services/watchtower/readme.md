# Watchtower

[Watchtower](https://github.com/nicholas-fedor/watchtower) will automatically update Podman/Docker images. This version is a fork of the original that has been discontinued.

It is used on this server to update any container orchestrated by Sablier (running via a `compose.yaml`).

Place `watchtower.container` in `~/.config/containers/systemd/watchtower` and `watchtower.timer` in `~/.config/systemd/user/`. Rather than run this service continually, it is called on-demand once per week.

I find Watchtower can be [buggy](https://github.com/nicholas-fedor/watchtower/issues/1292) in this setup and need to either find an alternative or the cause of these glitches.

## .env

Configure to your needs:

```dotenv
TZ=Europe/London
WATCHTOWER_RUN_ONCE=true
WATCHTOWER_INCLUDE_STOPPED=true
WATCHTOWER_LABEL_ENABLE=true
WATCHTOWER_NO_STARTUP_MESSAGE=true
WATCHTOWER_NOTIFICATION_LOG_STDOUT=true
WATCHTOWER_DISABLE_CONTAINERS=gts-infra,photoprism-infra,miniflux-infra,paperless-infra
WATCHTOWER_DEBUG=false
```