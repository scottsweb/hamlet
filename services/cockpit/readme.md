# Cockpit

[Cockpit](https://cockpit-project.org/) is a web-based graphical interface for servers, intended for everyone, but especially those who are new to Linux. It ships with uCore by default, so the only thing you need to do is enable the service:

```
sudo systemctl enable --now cockpit.service
```

## Firewall

I removed Cockpit from the Firewall as we send all traffic through Caddy instead. 

```bash
sudo firewall-cmd --permanent --zone=FedoraServer --remove-service=cockpit
```

Cockpit can then be configured to use a custom domain via [Caddy](https://github.com/scottsweb/hamlet/blob/main/services/caddy/caddy.container#L48-L51). Create a file at `sudo nano /etc/cockpit/cockpit.conf` and add the following:

```desktop
[WebService]
Origins = https://cockpit.example.com                  
ProtocolHeader = X-Forwarded-Proto
AllowUnencrypted = true
ForwardedForHeader = X-Forwarded-For
```

Cockpit can then be restarted: `sudo systemctl restart cockpit.service`.