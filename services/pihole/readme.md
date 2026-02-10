# Pi-hole

[Pi-hole](https://pi-hole.net/) is a software that blocks ads and trackers across your entire network. It can also provide DHCP for crappy routers that won't allow you to change the local DNS settings.

I currently run this container as root in order to provide DHCP to our network. The `pihole.container` file should be placed in the `/etc/containers/systemd` folder. When we are able to get a new router, I will switch Pi-hole to be rootless and move DHCP to the router.

## .env

Configure as required.

```dotenv
TZ=Europe/London

# I believe I will need this when running rootless
#PIHOLE_UID=1000
#PIHOLE_GID=1000

FTLCONF_webserver_api_password=!password
FTLCONF_dns_listeningMode=all
FTLCONF_webserver_port=8053,8054s
FTLCONF_dns_upstreams=1.1.1.1
FTLCONF_dns_interface=eno1
#FTLCONF_dns_dnssec=true

# Only required if you are using Pi-hole for DHCP
FTLCONF_dhcp_start=192.168.1.3
FTLCONF_dhcp_end=192.168.1.249
FTLCONF_dhcp_router=192.168.1.1
FTLCONF_dhcp_leaseTime=168
FTLCONF_dns_domain=lan
FTLCONF_dhcp_rapidCommit=true
```

## Firewall

```bash
sudo firewall-cmd --permanent --zone=FedoraServer --add-port=53/udp
sudo firewall-cmd --permanent --zone=FedoraServer --add-port=53/tcp
sudo firewall-cmd --permanent --zone=FedoraServer --add-port=67/udp
```

Use `sudo firewall-cmd --reload` to reload the firewall.

## Allow rootless containers to bind to port 53

To configure this for testing run `sudo sysctl net.ipv4.ip_unprivileged_port_start=53`, this can then be saved permanently by editing `sudo nano /etc/sysctl.conf` and adding:

```bash
net.ipv4.ip_unprivileged_port_start=53
```

## Disable internal DNS resolution

To avoid conflicts on port 53 and ensure the server has access to Pi-hole too, edit `sudo nano /etc/systemd/resolved.conf` and add:

```desktop
[Resolve]
DNSStubListener=no
```

Once modified run the following commands to apply the change:

```bash
sudo systemctl restart systemd-resolved
sudo systemctl restart NetworkManager
```

Or reboot.

## Migration of data to a new Pi-hole

Migrate your settings using the built in teleport feature. This will bring all your customisations, DNS servers etc to the new instance.

On this new server I then:

* Set a fixed IP address in [Cockpit](https://github.com/scottsweb/hamlet/tree/main/services/cockpit)
* Set the DNS in Cockpit to `127.0.0.1`
* Shutdown this server and the old server / Pi-hole
* Booted up this server