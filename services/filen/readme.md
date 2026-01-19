# Filen

[Filen](https://filen.io/r/79ba4782a74e4e48f30e6f7f41b99fc8) offers reliable zero-knowledge, client-side encrypted cloud storage you can trust. 

On this server Filen is used as a backup destination for Podman services and some of the ZFS pool data. For more details on how to configure this container visit the [official documentation](https://docs.filen.io/docs/cli/).

Place the timers and services in `~/.config/systemd/user/`, then run `systemctl --user daemon-reload`, followed by `systemctl --user enable --now filen-*.timer`.

## Example sync.json / sync-weekly.json file

```json
[
	{
		"local": "/mnt/myfiles",
		"remote": "/backups/myfiles",
		"syncMode": "localToCloud",
		"disableLocalTrash": true,
		"ignore": [
		],
		"excludeDotFiles": false,
		"alias": "backup-myfiles"
	},
]
```