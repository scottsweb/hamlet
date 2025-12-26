# HAM (Home Automation Machine)

This is version three of HAM ([version one](https://github.com/scottsweb/ham/tree/master), [version two](https://github.com/scottsweb/ham/tree/main)). The host machine is running [uCore](https://github.com/ublue-os/ucore) and uses a combination of [Podman](https://podman.io/) [Quadlets](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html) and [Compose](https://docs.podman.io/en/stable/markdown/podman-compose.1.html) files. 

This majority of the containers run rootless and for the few that don't, I will explain why in their individual readme files.

## Ignition

The Butane file here is very much a work in progress, the eventual goal is to have a completely replicable home server... but I also like the idea of having a minimal Butane file that acts as a simple starting point. That's what you'll find right now. The following instructions are for a bare metal install of [uCore](https://github.com/ublue-os/ucore).

1. Edit beep.butane with your own [keys](https://coreos.github.io/butane/getting-started/#writing-and-using-butane-configs) and [password](https://coreos.github.io/butane/examples/#using-password-authentication) for the `core` user
1. Install [Butane](https://coreos.github.io/butane/getting-started/) and create your [Ignition](https://coreos.github.io/ignition/) file `butane beep.butane -o beep.ign`
1. Serve the Ignition file on your local network, you can use something like [serve](https://github.com/vercel/serve)
1. Write the [CoreOS](https://www.fedoraproject.org/coreos/) `.iso` [to a USB stick](https://etcher.balena.io/)
1. Once you have booted up your PC, install CoreOS using the locally served Ignition file. You'll need to change the destination disk and IP address in this example:

```bash
sudo coreos-installer install /dev/nvme0n1 \
    --ignition-url http://192.168.1.2:3000/beep.ign \
    --insecure-ignition
```

The install will take some time and the machine will reboot a number of times. Eventually you'll be setup with a fresh copy of uCore, ready to customise.

The last thing to do is check for the updates using `sudo rpm-ostree update`.

References: [Fedora CoreOS documentation](https://docs.fedoraproject.org/en-US/fedora-coreos/bare-metal/), [uCore Server Setup](https://daniel.melzaks.com/guides/ucore-server-setup/)

## Secure Boot

To setup secure boot run `sudo mokutil --import /etc/pki/akmods/certs/akmods-ublue.der`. After a reboot you can check that it's working by using `sudo mokutil --sb-state`. On my machine (a Lenovo m90q), I also had to turn on secure boot in the BIOS, ensure it was in `user mode` and also set `Allow Microsoft 3rd party UEFI CA` to `ON`.

Reference: [Enabling Secure Boot for Linux on Lenovo Secured-core
PC’s](https://download.lenovo.com/pccbbs/mobiles_pdf/Enable_Secure_Boot_for_Linux_Secured-core_PCs.pdf)

## Encryption

The Butane/Ignition file encrypts the home partition of the drive using the [Trusted Platform Module](https://wiki.archlinux.org/title/Trusted_Platform_Module) (TPM2) and [LUKS](https://docs.fedoraproject.org/en-US/quick-docs/encrypting-drives-using-LUKS/#_encrypting_block_devices_using_dm_cryptluks). In many instances, this will be sufficient, but I also like to set a password and create a USB key for decryption.

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

# mount the USB drive on the server, us sudo dmesg or lsblk to find the /dev path
sudo mount -t auto /dev/sda /mnt/pendrive

# change the permissions of the boot-key
sudo chmod 600 /mnt/pendrive/boot-key

# add the boot-key as a method to unlock LUKS
sudo crypsudo cryptsetup luksAddKey /dev/nvme0n1p4 /mnt/pendrive/boot-key

# check the keys correctly in place
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
rd.luks.key=a54462a1-e264-1fa9-83b4-4e9efab84a33=/beep-boot:UUID=e3ec4e11-2d55-2cfb-82a9-b22ff935e21c 
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

### Create the ZFS pool manually

I had some issues setting up a mirrored pool in the [Cockpit](https://cockpit-project.org/) interface as my two disks are slightly different sizes, so created the pool manually:

```bash
sudo zpool create -o ashift=12 -o autotrim=on -o autoreplace=on -o autoexpand=on -O encryption=aes-256-gcm -O keyformat=raw -O keylocation=file:///var/mnt/pendrive/zfs-key -O compression=lz4 -O atime=off -O casesensitivity=insensitive -f -m /var/mnt/data data mirror /dev/disk/by-path/pci-0000:03:00.0-nvme-1 /dev/disk/by-path/pci-0000:02:00.0-nvme-1
```

I was then able to setup the individual filesystems with Cockpit (`http://localip:9090`), ensuring the correct permissions are set either in the UI or with `sudo chown $USER:$USER /var/mnt/data/myfiles` and `sudo chmod 755 /var/mnt/data/myfiles`.

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
