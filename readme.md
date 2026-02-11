# HAM (Home Automation Machine)

This is version three of HAM ([version one](https://github.com/scottsweb/ham/tree/master), [version two](https://github.com/scottsweb/ham/tree/main)). The host machine is running [uCore](https://github.com/ublue-os/ucore) and uses a combination of [Podman](https://podman.io/) [Quadlets](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html) and [Compose](https://docs.podman.io/en/stable/markdown/podman-compose.1.html) files. 

This majority of the containers run rootless and for the few that don't, I will explain why in their [individual readme files](#services).

## Ignition

The Butane file here is very much a work in progress, the eventual goal is to have a completely replicable home server... but I also like the idea of having a minimal Butane file that acts as a simple starting point. That's what you'll find right now. The following instructions are for a bare metal install of [uCore](https://github.com/ublue-os/ucore).

1. Edit `ucore.butane` with your own [keys](https://coreos.github.io/butane/getting-started/#writing-and-using-butane-configs) and [password](https://coreos.github.io/butane/examples/#using-password-authentication) for the `core` user
1. Edit `ucore.butane` and add a custom hostname (around line `32`)
1. Choose your uCore version by editing lines `47` and `65`: `ghcr.io/ublue-os/ucore:stable-nvidia`
1. Install [Butane](https://coreos.github.io/butane/getting-started/) and create your [Ignition](https://coreos.github.io/ignition/) file `butane ucore.butane -o ucore.ign`
1. Serve the Ignition file on your local network, you can use something like [serve](https://github.com/vercel/serve)
1. Write the [CoreOS](https://www.fedoraproject.org/coreos/) `.iso` [to a USB stick](https://etcher.balena.io/)
1. Turn off Secure Boot in the BIOS
1. Once you have booted up your PC, install CoreOS using the locally served Ignition file. You'll need to change the destination disk and IP address in this example:

```bash
sudo coreos-installer install /dev/nvme0n1 \
    --ignition-url http://192.168.1.2:3000/ucore.ign \
    --insecure-ignition
```

The install will take some time and the machine will reboot a number of times. Eventually you'll be setup with a fresh copy of uCore, ready to customise.

The last thing to do is check for updates using `sudo bootc update`.

References: [Fedora CoreOS documentation](https://docs.fedoraproject.org/en-US/fedora-coreos/bare-metal/), [uCore Server Setup](https://daniel.melzaks.com/guides/ucore-server-setup/)

## Secure Boot

To setup secure boot run `sudo mokutil --import /etc/pki/akmods/certs/akmods-ublue.der`. The utility will prompt for a password. The password will be used to verify this key is the one you meant to import. After the reboot and import you can check that it's all working with: `sudo mokutil --sb-state`. 

On my machine (a Lenovo M90q), after re-enabling secure boot in the BIOS, I also had to ensure it was in `user mode` and that `Allow Microsoft 3rd party UEFI CA` was set to `ON`.

Reference: [Enabling Secure Boot for Linux on Lenovo Secured-core
PC’s](https://download.lenovo.com/pccbbs/mobiles_pdf/Enable_Secure_Boot_for_Linux_Secured-core_PCs.pdf)

## Encryption

The Butane/Ignition file encrypts the home partition of the drive using the [Trusted Platform Module](https://wiki.archlinux.org/title/Trusted_Platform_Module) (TPM2) and [LUKS](https://docs.fedoraproject.org/en-US/quick-docs/encrypting-drives-using-LUKS/#_encrypting_block_devices_using_dm_cryptluks). In many instances, this will be sufficient, but I also like to set a password and create a USB key for decryption.

Use `lsblk --fs` to list all partitions and their UUIDs. Partitions using an FSTYPE of `crypto_LUKS` are your encrypted partitions.

References: [Unlocking a LUKS-encrypted partition on boot with an USB drive](https://blog.fidelramos.net/software/unlock-luks-usb-drive), [Unlocking a luks volume with a USB key](https://possiblelossofprecision.net/?p=300)

### Add a passphrase / password LUKS key

```bash
# export the current LUKS key into a temporary file
sudo clevis luks pass -d /dev/nvme0n1p4 -s 1 > tmpkey

# create a new key, using the temporary key file 
sudo cryptsetup luksAddKey /dev/nvme0n1p4 --key-file tmpkey

# test your new passphrase
sudo cryptsetup --verbose open --test-passphrase /dev/nvme0n1p4

# delete the temporary key file
sudo rm tmpkey
```

### Add a USB LUKS key

```bash
# create a key file on an external USB drive / memory pen (I did this on my laptop), the drive is formatted as ext4
dd if=/dev/urandom of=/run/media/user/key-drive/boot-key bs=4096 count=1 

# mount the USB drive on the server, use sudo dmesg or lsblk to find the /dev path
sudo mount -t auto /dev/sda /mnt/pendrive

# change the permissions of the boot-key
sudo chmod 600 /mnt/pendrive/boot-key

# add the boot-key as a method to unlock LUKS
sudo crypsudo cryptsetup luksAddKey /dev/nvme0n1p4 /mnt/pendrive/boot-key

# check the key is correctly installed
sudo cryptsetup luksDump /dev/nvme0n1p4
```

Now we need to make sure the USB key unlocks the drive at boot time:

```bash
# check the device ID for the USB drive / pen and also the partition you wish to unlock
ls -l /dev/disk/by-uuid

# edit the boot args using the default editor
rpm-ostree kargs --editor

# add the following to the file, with the IDs from above, in this example a544... is the root/boot partition and e3ec... the USB key
rd.luks.name=a54462a1-e264-1fa9-83b4-4e9efab84a33=root 
rd.luks.key=a54462a1-e264-1fa9-83b4-4e9efab84a33=/boot-key:UUID=e3ec4e11-2d55-2cfb-82a9-b22ff935e21c 
rd.luks.options=a54462a1-e264-1fa9-83b4-4e9efab84a33=keyfile-timeout=5s

# save these changes and reboot
```

### Remove the TPM2 LUKS key (optional)

If you don't want to tie your disk encryption to the TPM2 module, you can remove that key:

```bash
# check which slot has the clevis / TPM2 key
sudo cryptsetup luksDump /dev/nvme0n1p4

# remove the clevis / TMP2 key, with 1 being the slot found above
sudo clevis luks unbind -d /dev/nvme0n1p4 -s 1
```

## ZFS

### Enable the ZFS kernel module

```bash
echo zfs > /etc/modules-load.d/zfs.conf
```

### Create a USB encryption key for ZFS

```bash
# create the key file on same USB drive as before
dd if=/dev/urandom of=/mnt/pendrive/zfs-key bs=1 count=32
```

Auto mount the USB drive on each boot adding the following to `/etc/fstab`:

```bash
# the device UUID can be found with ls -l /dev/disk/by-uuid
UUID=e3ec4e11-2d55-2cfb-82a9-b22ff935e21c /mnt/pendrive auto ro,nofail 0 0
```

Check your changes to `/etc/fstab` before a reboot with `mount -a`.

### Create the ZFS pool manually

I had some issues setting up a mirrored pool in the [Cockpit](https://cockpit-project.org/) interface as my two disks are slightly different sizes, so created the pool manually:

```bash
sudo zpool create -o ashift=12 -o autotrim=on -o autoreplace=on -o autoexpand=on -O encryption=aes-256-gcm -O keyformat=raw -O keylocation=file:///var/mnt/pendrive/zfs-key -O compression=lz4 -O atime=off -O casesensitivity=insensitive -f -m /var/mnt/data data mirror /dev/disk/by-path/pci-0000:03:00.0-nvme-1 /dev/disk/by-path/pci-0000:02:00.0-nvme-1
```

I was then able to setup the individual filesystems within Cockpit (`http://localip:9090`), ensuring the correct permissions were set either in the UI or with `sudo chown $USER:$USER /var/mnt/data/myfiles` and `sudo chmod 755 /var/mnt/data/myfiles` after creation.

References: [Silverblue and Cockpit](https://www.youtube.com/watch?v=30ICVF0LRsY), [ZFS management with Cockpit](https://www.youtube.com/watch?v=A711MXlyFac)

### Unlock the ZFS pool at boot

Create a new service unit file at `/etc/systemd/system/zfs-load-key@.service`:

```desktop
[Unit]
Description=Load ZFS keys
DefaultDependencies=no
Before=zfs-mount.service
After=zfs-import.target
Requires=zfs-import.target
RequiresMountsFor=/mnt/pendrive/zfs-key

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/sbin/zfs load-key %I

[Install]
WantedBy=zfs-mount.service
```

And enable it for the `data` pool with `sudo systemctl daemon-reload`, followed by `sudo systemctl enable zfs-load-key@data`.

References: [How to auto load-key and mount natively encrypted ZFS Pool](https://forum.openmediavault.org/index.php?thread/40525-how-to-auto-load-key-and-mount-natively-encrypted-zfs-pool-no-luks/), [Mounting encrypted datasets automatically](https://github.com/openzfs/zfs/issues/8750#issuecomment-497500144), [Best practices for unlocking ZFS dataset](https://www.reddit.com/r/zfs/comments/w33bss/looking_for_best_practice_for_unlocking_encrypted/)

### Unmount the USB drive after boot

USB drives have a habit of wearing out, so I figured it is good to unmount the drive once everything is unlocked. If you want to do the same, create a new unit file at `/etc/systemd/system/zfs-unload-key-after-boot.service`:

```desktop
[Unit]
Description=Unmount ZFS key after boot
After=multi-user.target
Requires=local-fs.target

[Service]
Type=oneshot
ExecStart=/bin/umount /var/mnt/pendrive
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

And enable it with `sudo systemctl daemon-reload`, followed by `sudo systemctl enable zfs-unload-key-after-boot`.

### Scrub

`sudo systemctl enable --now zfs-scrub-monthly@data.timer` will enable a monthly scrub of the ZFS pool named `data`. A scrub will check the integrity of the stored data and repair if necessary. You may choose a custom time for the timer to run by editing it `sudo systemctl edit zfs-scrub-monthly@data.timer` and adding the following where indicated:

```desktop
# scrub first friday of each month at 13:00, verify with systemd-analyze calendar "Fri *-*-1..7 13:00:00" --iterations=3
[Timer]
OnCalendar=
OnCalendar=Fri *-*-1..7 13:00:00
```

Apply the changes with `sudo systemctl daemon-reload`.

I have found scrubbing to be an extremely intensive task and in tiny PC, hard drives can get very hot. I am currently attempting to tune the process via `zfs.conf`. Edit the file with `sudo nano /etc/modprobe.d/zfs.conf` and set the following options:

```bash
# sets the maximum scrub or scan read I/Os active to each device, default is 3
options zfs zfs_vdev_scrub_max_active=1

# the maximum amount of data that can be concurrently issued at once for scrubs and resilvers per leaf vdev,default is 16777216
options zfs zfs_scan_vdev_limit=8388608
```

After a reboot you can check the values have been applied by inspecting the relevant files in the `/sys/module/zfs/parameters/` folder.

### Trim

Trimming is a process that informs the SSD about which data blocks are no longer in use, allowing the SSD to manage its storage more efficiently.

Although I setup my pool and datasets/filesystems with `autotrim`, I also enabled a monthly trim process too: `sudo systemctl enable --now zfs-trim-monthly@data.timer`. Once again I edited the timer `sudo systemctl edit zfs-trim-monthly@data.timer` to run on a schedule that worked for me:

```desktop
# trim first thursday of each month at 13:00, verify with: systemd-analyze calendar "Thu *-*-1..7 13:00:00" --iterations=3
[Timer]
OnCalendar=
OnCalendar=Thu *-*-1..7 13:00:00
```

These changes are then applied with `sudo systemctl daemon-reload` and the status of the timer verified by running `sudo systemctl status zfs-trim-monthly@data.timer`.

### Sanoid

[Sanoid](https://github.com/jimsalterjrs/sanoid) is a open-source tool for automating ZFS filesystem snapshot management, making it easy to take, prune (delete old), and replicate snapshots. Sanoid is managed by editing `/etc/sanoid/sanoid.conf`.

```desktop
# datasets
[data]
    use_template = ignore
    process_children_only = yes

[data/myfiles]
    use_template = weekly

[data/backups]
    use_template = monthly

# templates
[template_ignore]
    autoprune = no
    autosnap = no
    monitor = no

[template_weekly]
    frequently = 0
    hourly = 0
    daily = 0
    weekly = 12
    monthly = 0
    yearly = 0
    weekly_wday = 1
    weekly_hour = 16
    weekly_min = 0
    autosnap = yes
    autoprune = yes
    monitor = yes
    capacity_warn = 80
    capacity_crit = 90

[template_monthly]
    frequently = 0
    hourly = 0
    daily = 0 
    weekly = 0
    monthly = 12
    yearly = 0
    monthly_mday = 1
    monthly_hour = 15
    monthly_min = 0
    autosnap = yes
    autoprune = yes
    monitor = yes
    capacity_warn = 80
    capacity_crit = 90
```

Debug the configuration with `sudo sanoid --take-snapshots --readonly --debug` and when you are ready, enable the Sanoid timer `sudo systemctl enable --now sanoid.timer`.

Reference: [Avoiding data disasters with Sanoid](https://opensource.com/life/16/7/sanoid)

### Syncoid

[Syncoid](https://github.com/jimsalterjrs/sanoid?tab=readme-ov-file#syncoid) enables fast and asynchronous replication of ZFS filesystems (datasets) between servers. I don't have a secondary server (yet), so will investigate this later.

Reference: [Backups with Sanoid/Syncoid](https://github.com/ublue-os/ucore#backups-with-sanoidsyncoid)

## Podman

[Podman](https://podman.io/) is used to orchestrate pods and container services on the server.

### Enable the Podman socket

The socket allows containers and tools to manage Podman using a Docker compatible API.

```bash
# rootful (if needed)
sudo systemctl enable --now podman.socket

# core user
systemctl --user enable --now podman.socket
```

### Enable Podman auto updates 

Any [Quadlet](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html) with `AutoUpdate=registry` will be [automatically updated](https://docs.podman.io/en/latest/markdown/podman-auto-update.1.html) on a schedule of your choosing:

```bash
# rootful (if needed)
sudo systemctl enable --now podman-auto-update.timer

# core user
systemctl --user enable --now podman-auto-update.timer
```

To choose the time and frequency of the updates, you can edit the timer with `sudo systemctl edit podman-auto-update.timer` for rootful containers or `systemctl --user edit podman-auto-update.timer` for rootless. Drop your desired schedule where indicated and save. For example:

```desktop
# auto update every wednesday at 12:00
[Timer]
OnCalendar=
OnCalendar=Wed 12:00:00
```

These changes are then applied with `sudo systemctl daemon-reload` / `systemctl --user daemon-reload` and the status of the timer verified by running `sudo systemctl status podman-auto-update.timer` / `systemctl --user status podman-auto-update.timer`.

### Keep Podman and the firewall in sync

Podman and firewalld [can sometimes conflict](https://github.com/ublue-os/ucore?tab=readme-ov-file#podman-and-firewalld) such that a `firewall-cmd --reload` removes firewall rules generated by podman. This can be fixed by enabling the following service:

```bash
sudo systemctl enable netavark-firewalld-reload.service
```

### Auto restart for compose containers

By default, any Podman containers started via a `compose.yaml` file and with the `restart: always` policy will not restart on a reboot. This functionality is enabled with the following service:

```bash
# rootful (if needed)
sudo systemctl enable --now podman-restart.service

# core user
systemctl --user enable --now podman-restart.service
```

## Services

* [Caddy](https://github.com/scottsweb/hamlet/tree/main/services/caddy)
* [Cockpit](https://github.com/scottsweb/hamlet/tree/main/services/cockpit)
* [Filen](https://github.com/scottsweb/hamlet/tree/main/services/filen)
* [Glance](https://github.com/scottsweb/hamlet/tree/main/services/glance)
* [Gluetun](https://github.com/scottsweb/hamlet/tree/main/services/gluetun)
* [GoToSocial](https://github.com/scottsweb/hamlet/tree/main/services/gotosocial)
* [Home Pod (Home Assistant and supporting services)](https://github.com/scottsweb/hamlet/tree/main/services/home)
* [Jellyfin](https://github.com/scottsweb/hamlet/tree/main/services/jellyfin)
* [Mazanoke](https://github.com/scottsweb/hamlet/tree/main/services/mazanoke)
* [Miniflux](https://github.com/scottsweb/hamlet/tree/main/services/miniflux)
* [NocoDB](https://github.com/scottsweb/hamlet/tree/main/services/nocodb)
* [Node-RED](https://github.com/scottsweb/hamlet/tree/main/services/node-red)
* [Ollama](https://github.com/scottsweb/hamlet/tree/main/services/ollama)
* [Paperless](https://github.com/scottsweb/hamlet/tree/main/services/paperless)
* Penpot (coming soon)
* [PhotoPrism](https://github.com/scottsweb/hamlet/tree/main/services/photoprism)
* [Pi-hole](https://github.com/scottsweb/hamlet/tree/main/services/pihole)
* [Sablier](https://github.com/scottsweb/hamlet/tree/main/services/sablier)
* [Samba](https://github.com/scottsweb/hamlet/tree/main/services/samba)
* [Shiori](https://github.com/scottsweb/hamlet/tree/main/services/shiori)
* [Watchtower](https://github.com/scottsweb/hamlet/tree/main/services/watchtower)
* [Web](https://github.com/scottsweb/hamlet/tree/main/services/web)
* WireGuard (wg-easy)
* [Wolf](https://github.com/scottsweb/hamlet/tree/main/services/wolf)

## Other tips

### Firmware upgrades

```bash
# see what's needed
fwupdmgr get-upgrades

# do the upgrades
sudo fwupdmgr update
```

### Rsync to different machine

```bash
# remove --dry-run when ready to copy
rsync -vr --delete --progress /media/sloan/data/ core@192.168.1.2:/mnt/data/ --dry-run
```

### Temperature checks

```bash
# list nvme drives
sudo nvme list

# check temperature of those indexed above
sudo nvme smart-log /dev/nvme1n1 | grep -i '^temperature'

# cpu temp
cat /sys/class/thermal/thermal_zone*/temp

# nvidia gpu temp
nvidia-smi -q -a | grep -i "temp"
```

### Debugging systemd container files

```bash
# rootful
sudo /usr/lib/systemd/system-generators/podman-system-generator -dryrun

# core user
/usr/libexec/podman/quadlet --dryrun -user
```

### Tuned

I haven't found any benefit from tuned yet... 

```bash
# enable
sudo systemctl enable --now tuned

# vidw active profile
tuned-adm active

# list all profiles
tuned-adm list

# switch profile
sudo tuned-adm profile powersave
```

### Additional SSH users

Keys can be added to `~/.ssh/authorized_keys.d/` as individual files.