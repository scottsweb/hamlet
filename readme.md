# HAM (Home Automation Machine)

This is version three of HAM ([version one](https://github.com/scottsweb/ham/tree/master), [version two](vhttps://github.com/scottsweb/ham/tree/main)). The host machine is running [uCore](https://github.com/ublue-os/ucore) and uses a combination of [Podman](https://podman.io/) [Quadlets](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html) and [Compose](https://docs.podman.io/en/stable/markdown/podman-compose.1.html) files. 

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

