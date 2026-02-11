# Utilities

Services (with timers) to automate certain aspects of the server. All timers and services should be placed in `~/.config/systemd/user/`.

* `container-backup.service/.timer` — Backup container data to the ZFS pool
* `duckdns.service/.timer` — Update [DuckDNS](https://www.duckdns.org/) (update the URL and token in the `.service` file to your own)
