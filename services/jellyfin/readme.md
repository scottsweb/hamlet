# Jellyfin

[Jellyfin](https://jellyfin.org/) is the volunteer-built media solution that puts you in control of your media. Stream to any device from your own server, with no strings attached. Your media, your server, your way.

## .env

Tweak these settings to your preference: 

```bash
TZ=Europe/London
JELLYFIN_PublishedServerUrl=https://jellyfin.example.com
```

## Firewall

Port `7359` is used for auto client discovery of Jellyfin on the network.

```bash
sudo firewall-cmd --permanent --zone=FedoraServer --add-port=7359/udp
```

Use `sudo firewall-cmd --reload` to reload the firewall.
