# Wolf / Games On Whales

[Wolf](https://games-on-whales.github.io/) is an open source streaming server for [Moonlight](https://moonlight-stream.org/) that allows you to share a single server with multiple remote clients in order to play games using services like [Steam](https://store.steampowered.com/), [RetroArch](https://www.retroarch.com/) and [Lutris](https://lutris.net/).

The container is run as root (although it will apparently run rootless too) and `wolf.container` should be placed in the `/etc/containers/systemd` folder. Alongside the container you will need to create an Nvidia driver volume as [covered in the documentation](https://games-on-whales.github.io/wolf/stable/user/quickstart.html). In fact, the documentation on the Wolf site is the best place to start as it can be a tricky service to configure depending on your hardware and stack. 

## .env

[Configure](https://games-on-whales.github.io/wolf/stable/user/configuration.html) as required:

```dotenv
#WOLF_LOG_LEVEL=DEBUG
WOLF_STOP_CONTAINER_ON_EXIT=TRUE
NVIDIA_DRIVER_VOLUME_NAME=nvidia-driver-vol
WOLF_RENDER_NODE=/dev/dri/renderD129
TZ=Europe/London
```

## Enable the Nvidia persistence service (put /dev/nvidia-modeset in place)

```bash
sudo systemctl enable --now nvidia-persistenced.service
```

## Enable uinput in the kernel

`sudo modprobe uinput` to enable it right now and then `sudo echo uinput > /etc/modules-load.d/uinput.conf` to make it persist after reboots. 

## Fix input group in uCore

```bash
grep -E '^input:' /usr/lib/group | sudo tee -a /etc/group

# if running rootless, this will need to be your user instead of root
sudo usermod -aG input root 
```

## Enable controller support

See [Wolf documentation for latest udev rules](https://games-on-whales.github.io/wolf/stable/user/quickstart.html#_virtual_devices_support).

Reference: [udev](https://wiki.archlinux.org/title/Udev)

## TuneD tweaking with Wolf and Steam

Under `/utilites/` you will find `wolf-watcher` and `wolf-watcher.service`. This script monitors Podman for the launch of the Steam container and adjusts the [TuneD](https://tuned-project.org/) profile accordingly. The service unit file should be placed in `/etc/systemd/system/` and enabled with:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now wolf-watcher.service
```

The bin/script can be placed where ever you like.

## ntsync

`ntsync` is a Linux kernel module designed to significantly improve the performance of Windows applications and games running on Linux via Wine or Proton. It should help with 1% lows and stuttering.Enable the kernel module with `sudo modprobe ntsync` and make it permanent on future reboots:

```
echo "ntsync" | sudo tee /etc/modules-load.d/ntsync.conf
```

The device `/etc/ntsync` can then be passed into Wolf through your `config.toml`: `devices = ['/dev/ntsync:/dev/ntsync']`.