# Home Pod

This is a collection of services run for Home Assistant, including ESPHome, MQTT and Music Assistant.

## Home Assistant

[Home Assistant](https://www.home-assistant.io/) is an open source home automation app that puts local control and privacy first. Powered by a worldwide community of tinkerers and DIY enthusiasts. 

### Nmap Tracker in a rootless Podman container

The `nmap` scanner has to run in unprivileged mode. To do this modify the Nmap Tracker options in the Home Assistant UI and add `--unprivileged` to the raw configurable scan options.

### Home Assistant Connect ZBT-1 / USB with rootless Podman 

The container needs access to USB which is a little tricky using Podman. You need to create a udev rule that changes the group of the USB device when its plugged in so the container can access it.

Check if your rootless containers are able to use devices with `getsebool -a | grep container_use_devices`. If this setting is `off`, turn it on with:

```bash
sudo setsebool -P container_use_devices true
```

Get the device ID to pass into the container with `ls -l /dev/serial/by-id/` and update `homeassistant.container` if required. 

Next add the core user to the `dialout` group:

```bash
# add core user to dialout group
sudo grep -E '^dialout:' /usr/lib/group | sudo tee -a /etc/group 
sudo usermod -aG dialout core
```

You will then need to create a udev rule for the USB device — `sudo nano /etc/udev/rules.d/99-skyconnect.rules`:

```bash
SUBSYSTEM=="tty", ATTRS{idVendor}=="10c4", ATTRS{idProduct}=="ea60", GROUP="dialout", MODE="0660"
```

The vendor and product ID can be found with `lsusb`. 

Finally, apply these changes:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

`ls -l /dev/ttyUSB0` should show the device is part of the `dialout` group. You can also test the rules are firing with `sudo udevadm test /dev/ttyUSB0`.

References: [Access USB from rootless container](https://bugzilla.redhat.com/show_bug.cgi?id=1770553), [udev rule tips](https://gist.github.com/edro15/1c6cd63894836ed982a7d88bef26e4af), [Using Yubikey for GPG & SSH in WSL](https://github.com/maxkagamine/dotfiles?tab=readme-ov-file#using-yubikey-for-gpg--ssh-in-wsl)

### Tracking containers with Home Assistant

The [Monitor Docker Home Assistant component](https://github.com/ualex73/monitor_docker) is useful for managing and monitoring containers from within Home Assistant.

## Music Assistant

[Music Assistant](https://www.music-assistant.io/) is a music library manager for your offline and online music sources which can easily stream your favourite music to a wide range of supported players and be combined with the power of Home Assistant!

### Firewall

```bash
sudo firewall-cmd --permanent --zone=FedoraServer --add-port=8097/tcp
sudo firewall-cmd --reload
```

## Useful links

* [LaMetric icon reference](https://developer.lametric.com/icons)