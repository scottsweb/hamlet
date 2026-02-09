# Sablier

[Sablier](https://github.com/sablierapp/sablier) s a free and open-source software that scales your workloads on demand. It provides integrations with multiple reverse proxies (like [Caddy](https://github.com/scottsweb/hamlet/tree/main/services/caddy)) and allows for different loading strategies.

On this server, certain services are scaled up and down on demand. This covers any service using a `compose.yaml` file (e.g. [NocoDB](https://github.com/scottsweb/hamlet/tree/main/services/nocodb)).

## .env

No .env file is required for this project, but one is created in case that changes in future.

## data/sablier.yaml

Example Sablier configuration:

```yaml
provider:
  name: podman
  autostoponstartup: true
sessions:
  # The default session duration (default 5m)
  default-duration: 5m
  # The expiration checking interval. 
  # Higher duration gives less stress on CPU. 
  # If you only use sessions of 1h, setting this to 5m is a good trade-off.
  expiration-interval: 30s
#logging:
#  level: debug
strategy:
  dynamic:
    custom-themes-path:
    # Show instances details by default in waiting UI
    show-details-by-default: false
    # Default theme used for dynamic strategy (default "hacker-terminal")
    default-theme: ghost
    # Default refresh frequency in the HTML page for dynamic strategy. This is important, see: https://github.com/sablierapp/sablier/issues/720#issuecomment-3510286330
    default-refresh-frequency: 6s
  blocking:
    # Default timeout used for blocking strategy (default 1m)
    default-timeout: 1m
```

