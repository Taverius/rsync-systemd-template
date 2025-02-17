# rsync-systemd-template
 Template for Rsync backups to a (network) remote as systemd units.

## Usage

1. Make a copy of `rsync-template.service` and `rsync-template.timer`, replacing `template` with some useful identifier.
   * If you're going to have just the one, `backup` is fine, but otherwise a sanitized version of the path, with slashes replaced by `-` and other non-alphanumeric characters replaced by `_` is saner.
   * The latter means a path like `/some/custom-path/name` leads to `rsync-some-custom_path-name.(service|timer)`
2. Edit the unit files and set all the values contained in `<angle brackets>`.
3. Copy them to `/etc/systemd/system/`.

### Service

[`[Unit]`](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html)

* `Description=` Be kind to future you and set it to something that explains at a glance what's getting backed up and where, like `/some/path Backup via Rsync to my-awesome-remote.domain`
* `Wants=` Set it to the name of the timer unit.
* `ConditionDirectoryNotEmpty=` If the local path is in the root filesystem you don't need this line, otherwise set it to the local path to be backed up to ensure rsync doesn't get called on it in case it fails to mount for whatever reason.
  If your mount is itself a systemd unit you can instead append that unit's name to `After=`, but this also works for things that don't get one, like nested `btrfs` subvolumes or certain ways to use`zfs`.
  It also lets you deploy the service and populate the path after.

[`[Service]`](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html)

* `User=` & `Group=` Change as needed.
* `CPUSchedulingPolicy=` & `IOSchedulingClass=` In rare case you might want to give the service a higher priority. Check [systemd.exec](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html).

* `RestartMaxDelaySec=` & `RestartSteps=` if your distribution ships an older systemd it will ignore these 2 lines, and say so in the logs. In which case you can remove them and extend `RestartSec=` to a larger value if needed.
  If you're running the timer often, for example `hourly`, you can also just remove all the `Restart` lines.

* `Environment=`

  * `PASS=` Its not a great idea to have your remote password readable by anyone who can read your logs, so store it somewhere only your `User=` and `Group=` can access. Just `echo password > password-file`, and make sure to `chmod o-r password-file` as `rsync` will refuse to run if the password-file can be read by others.

  * `LOCAL=` Its your local path. Don't forget the trailing slash or it'll sync the directory too.

  * `REMOTE=` You remote's definition. For some remotes you might need to make sure the path already exists.

  * `ARGS=` You'll want to customize this according to what you're backing up and where.
    If you're backing up already compressed files, for example, `--compress` just eats cpu for no benefit.
    On the other hand, with a modern remote and uncompressed source files over a slow link you might want to specify higher compression with `--compress-level=NUM`, check `man rsync`.
    Consider if you need `--one-file-system`, for example to exclude `btrfs` subvolumes.
    `--archive` might seem tempting but if you're backing up system files owned by root, keep in mind its not a good idea for a remote to allow a client to set files (potentially executable) as owned by root.

    * it also implies `--links` which you might not want, think about removing it with `--no-l` and using `--safe-links` instead, if at all.
    * If your remote filesystem supports extended attributes, `--fake-super` stores root attributes as extended attributes.
    * You might also want to add `--acls` and `--xattrs`.

    If your remote doesn't support snapshots you might want to remove `--delete-during`, though keep in mind this doesn't add much safety; if that's the situation you're in, you're much better off backing up to a local `borg` or `restic` repository and `rsync`'ing that over instead, or just backing up with those tools directly to your remote if you can.

* `ExecStart=` Check the path is correct for your system with `which rsync`.

### Timer

[`[Unit]`](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html)

* `Description=` As per the service definition, make it descriptive.

[`[Timer]`](https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html) 

* `Unit=` make this point to the service file.
* `OnCalendar=` See [systemd.timer](https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html) and [systemd.time](https://www.freedesktop.org/software/systemd/man/latest/systemd.time.html) on possible definitions. Examples are `hourly`, `daily`, `05:45`, `Sunday 03:00`.
* `OnBootSec=` Depending on how demanding the the backup job is, you might want to increase the delay between boot and the timer triggering.
* `Persistent=` This makes sure the timer gets run even if the system was off when it would have triggered. If you're running it very often, for example `hourly` then you can remove this.
