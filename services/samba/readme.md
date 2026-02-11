# Samba

[Samba](https://www.samba.org/) is the most feature-rich Open Source implementation of the SMB and Active Directory protocols for Linux and UNIX-like systems. The container is provided by [Dockur](https://github.com/dockur/samba).

## .env

```dotenv
TZ=Europe/London
UID=1000
GID=1000
```

## Configuration

`smb.conf` example:

```desktop
[global]
    workgroup = WORKGROUP
    server string = ucore
    server role = standalone server
    server services = -dns, -nbt
    server signing = default
    server multi channel support = yes
    dns proxy = no

    security = user
    guest account = nobody
    pam password change = yes
    map to guest = bad user
    usershare allow guests = yes
    winbind scan trusted domains = yes
    hosts allow = 192.168.1.0/24
    hosts deny = 0.0.0.0/0
    force user = root
    force group = root

    idmap config * : backend = tdb
    idmap config * : range = 1000000-2000000

    vfs objects = fruit streams_xattr
    fruit:metadata = stream
    fruit:model = MacSamba
    fruit:posix_rename = yes
    fruit:veto_appledouble = no
    fruit:wipe_intentionally_left_blank_rfork = yes
    fruit:delete_empty_adfiles = yes
    fruit:time machine = yes

    load printers = no
    printing = bsd
    printcap name = /dev/null
    disable spoolss = yes

    log level = 0

[myfiles-read-only]
    path = /mnt/myfiles
    browsable = yes
    read only = yes
    guest ok = yes
    veto files = /._*/.apdisk/.AppleDouble/.DS_Store/.TemporaryItems/.Trashes/desktop.ini/ehthumbs.db/Network Trash Folder/Temporary Items/Thumbs.db/
    delete veto files = yes

[backups]
    path = /mnt/backups
    browsable = yes
    read only = no
    guest ok = no
    valid users = core
    write list = core
    veto files = /._*/.apdisk/.AppleDouble/.DS_Store/.TemporaryItems/.Trashes/desktop.ini/ehthumbs.db/Network Trash Folder/Temporary Items/Thumbs.db/
    delete veto files = yes
```

`users.conf` example:

```bash
# username:UID:groupname:GID:password:homedir
core:1000:core:1000:!password
```

The password is a clear text password of your choosing.

Reference: [Setting up Samba on Fedora](https://docs.fedoraproject.org/en-US/quick-docs/samba/)