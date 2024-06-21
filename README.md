# borgsnap - Backups using ZFS snapshots, borg, and (optionally) rsync.net

This is a simple script for doing automated daily backups of ZFS filesystems.
It uses ZFS snapshots, but could easily be adaptable to other filesystems,
including those without snapshots.

> This fork provides an user.scripts template for unraid and easy install instructions.
>
> **Notable Changes:**
> * Switch from repokey (HMAC-SHA256) to repokey-blake2 (BLAKE2b-256) for performance and security reasons  
>   Note: this should not affect older repos when migrating from an older fork
> * Implement BACKUP_TYPE  
>   Besides `zfs`, `plain` is supported so you can backup any folder (independant of any snapshots).
> * Unraid User.Scripts compatibility  
>   This fork adds a `script` files that is automatically picked up by unraid user scripts and handles the backup with a clean configuration option from Unraid UI
> * Self-Update (WIP)  
>   This fork added an `update` script that updates the tool if changes are available upstream.
> * MonoRepo  
>   This fork offers the option to backup all paths / snapshots into a single
>   borg repo (instead of multiple). This has the advantage of a better
>   deduplication and a security benefit since the key would be reused for
>   multiple repos and since borg is using AES-CTR, one should not use the same
>   key for different repos
>
> Previous Forks:
> 
> 1. https://github.com/scotte/borgsnap
> 2. https://github.com/ManuelSchmitzberger/borgsnap
> 3. https://github.com/jortan/borgsnap
> 4. https://github.com/michael-d-stone/borgsnap

## What is BORG

[Borg](https://www.borgbackup.org/) has excellent deduplication, so unchanged
blocks are only stored once. Borg backups are encrypted and compressed
(borgsnap uses lz4).

Unlike tools such as Duplicity, borg uses an incremental-forever model so you
never have to make a full backup more than once. This is really great when
sending full offsite backups might take multiple days to upload.

Borgsnap has optional integration with rsync.net for offsite backups. rsync.net
offers a [cheap plan catering to borg](http://www.rsync.net/products/attic.html).
As rsync.net charges nothing in transfer fees nor penalty fees for early
deletion, it's a very appealing option that is cheaper than other cloud storage
providers such as AWS S3 or Google Cloud Storage once you factor in transfer
costs and fees for deleting data.

Borgsnap automatically purges snapshots and old borg backups (both locally
and remotely) based on retention settings given in the configuration.
There is also the possibility to backup an already existing snapshot.

This assumes borg version 1.0 or later.

## How It Works

Borgsnap is pretty simple, it has the following basic flow:

+ Read configuration file and encryption key file
+ Validate output directory exists and a few other basics
+ For each ZFS filesystem do the following steps:
  + Initialize borg repositories if local one doesn't exist
  + Take a ZFS snapshot of the filesystem (recursively if enabled)
  + Run borg for the local output if configured
  + Run borg for the rsync.net output if configured
  + Delete old ZFS snapshots (recursively if enabled)
  + Prune local borg if configured and needed
  + Prune rsync.net borg if configured and needed

That's it!

If things fail, it is not currently re-entrant. For example, if a ZFS snapshot
already exists for the day, the script will fail\*.  This could use a bit of
battle hardening, but has been working well for me for several months already.

\* If the script does fail, you can use "tidy" option.  This will make best
effort to remove any mountpoints, delete today's zfs snapshots and borg
archives, allowing borgsnap to be run again that day.  This was added mostly
for test/dev purposes and may not work as intended!

> Finally, these things are probably obvious, but: Make sure your local backups
> are on a different physical drive than the data you are backing up and don't
> forget to do remote backups, because a local backup isn't disaster proofing
> your data.

## Installation

### Download

```bash
git clone https://github.com/mirisbowring/borgsnap.git 
```

Or on UNRAID ([User Scripts Plugin](https://forums.unraid.net/topic/48286-plugin-ca-user-scripts/) must be installed)

```bash
git clone https://github.com/mirisbowring/borgsnap.git /boot/config/plugins/user.scripts/scripts/borgsnap
```

### Updates

Just navigate into the cloned repository and execute the following:

```bash
/bin/bash update
```

### Generate Key

```
pwgen 128 1 > /path/to/my/super/secret/myhost.key
```

Or on UNRAID

```bash
openssl rand -base64 64 | openssl enc -A -base64 > /boot/config/plugins/user.scripts/scripts/borgsnap/backup.key
```

### Config

You could copy `sample.conf` or `sample_rsync.conf` to e.g. `backup.conf` and
adapt the values to your needs.

On Unraid you can see the script now on the WEB-UI under `/Settings/Userscripts`.
Click on the gear next to `borgsnap` and select `Edit Script`. Now you can adapt the values within the WebUI.

> Even if a Variable is not used, set it to `""` or the script will fail.

You may want to create a cronjob for a daily schedule!

**VALUES**

| Key | Default Value | Description |
| --- | ------------- | ----------- |
| FS |  | List file ZFS filesystems to backup. E.g. `FS="zroot/root zroot/home zdata/data"` |
| LOCAL |  | If specified (not ""), directory for borg backups. Backups will be stored in subdirectories of pool and filesystem. E.g. `zroot/root` would be stored in `/backup/borg/zroot/root`. **This directory must be created prior to running borgsnap.** |
| BASEDIR  | $HOME | This location is used for borg caching. On Unraid this should be a persisted folder (the default /root is not persisted across reboots). |
| LOCAL_READABLE_BY_OTHERS | false | Make borg repo readable by non-root |
| LOCALSKIP | false | If specified, borgsnap will skip local backup destinations and only issue backup commands to REMOTE destination |
| RECURSIVE | true | Create recursive ZFS snapshots for all child filsystems beneath all filesystems specified in "FS".  All child filesystems will be mounted for borgbackup. |
| COMPRESS | zstd | hoose compression algo for Borg backups.  Default for borgbackup is lz4, default here is zstd (which applies zstd,3) |
| CACHEMODE | ctime,size,inode | How borg will detect changes [see here](https://borgbackup.readthedocs.io/en/stable/usage/create.html#description) |
| REMOTE |  | If specified (not ""), remote connect string and directory. Only rsync.net has been tested. The remote directory (myhost in the example) will be created if it does not exist. E.g. `XXXX@YYYY.rsync.net:myhost`|
| REMOTE_BORG_COMMAND | borg1 | "borg1" for rsync, otherwise "borg" as appropriate |
| PASS |  | Path to a file containing a single line with the passphrase for borg encryption. [See Generate Key](#generate-key) |
| MONTH_KEEP | 1 | Number of month backups to keep. |
| WEEK_KEEP | 4 | Number of week backups to keep. |
| DAY_KEEP | 7 | Number of day backups to keep. |
| PRE_SCRIPT |  | will run before taking a snapshot for each dataset.  The [example provided](https://github.com/mirisbowring/borgsnap/blob/master/sample_prescript.sh) demonstrates how to run a command only for a specific dataset. Specify the full path to the script. |
| POST_SCRIPT |  | will run after taking a snapshot for each dataset.  The [example provided](https://github.com/mirisbowring/borgsnap/blob/master/sample_postscript.sh) demonstrates how to run a command only for a specific dataset. Specify the full path to the script. |
| NO_CONFIG | false | (Mostly needed for UNRAID) use this to disable config file check for borgsnap (only needed when using this script template) |
| BACKUP_TYPE |  | `zfs` for ZFS filesystems / `plain` for standard paths | 
| BORG_REPO |  | Name the monolithic borg repo you would like to use. Every Filesystem will be stored within this repo but as separate archive. E.g. `photos_month-20240621`

### Command Line

After configuring this script, you can execute borgsnap with the following arguments.

```
usage: borgsnap <command> <config_file> [<args>]

commands:
    run             Run backup lifecycle.
                    usage: borgsnap run <config_file>

    snap            Run backup for specific snapshot.
                    usage: borgsnap snap <config_file> <snapshot-name>
					
    tidy            Unmount and remove snapshots/local backups for today
                    usage: borgsnap tidy <config_file>
					
                    Added for test/dev purposes, may not work as intended!

                    Note: this will unmount all snapshots mounted by borgsnap
                    including other running instances.	
```

else you could use the preconfigured execution script for the user scripts plugin.

```bash
./script
```

## Restoring files

Borgsnap doesn't help with restoring files, it just backs them up. Restorations
are done directly from borg (or ZFS snapshots if it's a simple file deletion to
be restored). A backup that can't be restored from is useless, so you need to
test your backups regularly.

For Borgsnap, there are three ways to restore, depending on why you need to:

+ Use the local ZFS snapshot (magic .zfs directory on each ZFS filesystem).
This is the way to go if you simply deleted a file and there is no hardware
failure.

+ Use the local borg repository. If there is data loss on the ZFS filesystem,
but the backup drive is still good, use "borg mount" to mount up the directory
and restore files. See example below.

+ Use the remote borg repository. As with a local repository, use "borg mount"
to restore files from rsync.net.

The borgwrapper script in this repository can be used to set BORG_PASSPHRASE
from the borgsnap configuration file, making this slightly easier.

### Restoration Examples

Note: Instead of setting BORG_PASSPHRASE as done here, with an exported
environment variable, you can paste it in interactively.

Also note that borgsnap does backups directly from the ZFS snapshot, using
the magic .zfs mount point, hence the borg snapshot preserves this directory
structure. Don't worry, borg is still deduplicating files, even though the
directory changes each time. Also, don't panic if you do "ls /mnt" and don't
see anything - try "ls -a /mnt" or you might miss seeing that .zfs directory.

```
$ sudo -i

# export BORG_PASSPHRASE=$(</path/to/my/super/secret/myhost.key)

# borg list /backup/borg/zroot/root
week-20171008                        Sun, 2017-10-08 01:07:29
day-20171009                         Mon, 2017-10-09 01:07:54
day-20171010                         Tue, 2017-10-10 01:07:48
day-20171011                         Wed, 2017-10-11 01:07:57

# borg mount /backup/borg/zroot/root::day-20171011 /mnt

# ls /mnt/.zfs/snapshot/day-20171011/
backup	bin   etc  home	 lib64  proc  root  sbin  tmp  var

# borg umount /mnt
```

Restoring from rsync.net is nearly the same, just a change in the path, and
passing --remote-path=borg1 since we are using a modern borg version:

```
# borg mount --remote-path=borg1 XXXX@YYYY.rsync.net:myhost/zroot/root::day-20171011 /mnt
```

I used "borg mount" above, where we would, simply "cp" the files out. See
the borg manpages to read about other restoration options, such as
"borg extract".

And finally, using the borgwrapper script, which will set BORG_PASSPHRASE for
you:
```
# borgwrapper /path/to/my/borgsnap.conf list /backup/borg/zroot/root
[...]
```
