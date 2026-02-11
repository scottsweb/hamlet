# WireGuard Easy

[WireGuard Easy](https://github.com/wg-easy/wg-easy) provides an easy way to run WireGuard VPN, with a simple web UI.

## .env

```
PASSWORD_HASH=!password
TZ=Europe/London
WG_DEFAULT_DNS=10.89.0.1
WG_ALLOWED_IPS=192.168.1.0/24, 10.89.0.0/24, 10.8.0.0/24
WG_HOST=example.com
```

## Firewall

```bash
sudo firewall-cmd --permanent --zone=FedoraServer --add-service=wireguard
```

Use `sudo firewall-cmd --reload` to reload the firewall.

Reference: [How to find a list of services supported by firewalld](https://access.redhat.com/solutions/7045355)

## Kernel modules

WireGuard requires some additional Kernel modules. Modify `sudo nano /etc/modules-load.d/wireguard.conf` and add the following:

```bash
ip_tables

# add these too if your using another OS (they are already enabled on uCore)
wireguard
nft_masq
```

Modules can be temporarily loaded with `modprobe ip_tables` and `lsmod` can be used to check which modules are active. The modules above should be auto-loaded on future reboots.