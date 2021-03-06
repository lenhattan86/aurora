# Recovering from a Scheduler Backup

**Be sure to read the entire page before attempting to restore from a backup, as it may have
unintended consequences.**

## Summary

The restoration procedure replaces the existing (possibly corrupted) Mesos replicated log with an
earlier, backed up, version and requires all schedulers to be taken down temporarily while
restoring. Once completed, the scheduler state resets to what it was when the backup was created.
This means any jobs/tasks created or updated after the backup are unknown to the scheduler and will
be killed shortly after the cluster restarts. All other tasks continue operating as normal.

Usually, it is a bad idea to restore a backup that is not extremely recent (i.e. older than a few
hours). This is because the scheduler will expect the cluster to look exactly as the backup does,
so any tasks that have been rescheduled since the backup was taken will be killed.

Instructions below have been verified in [Vagrant environment](../getting-started/vagrant.md) and with minor
syntax/path changes should be applicable to any Aurora cluster.

Follow these steps to prepare the cluster for restoring from a backup:

##  Preparation

* Stop all scheduler instances.

* Pick a backup to use for rehydrating the mesos-replicated log. Backups can be found in the
directory given to the scheduler as the `-backup_dir` argument. Backups are stored in the format
`scheduler-backup-<yyyy-MM-dd-HH-mm>`.

* If running the Aurora Scheduler in HA mode, pick a single scheduler instance to rehydrate.

* Locate the `recovery-tool` in your setup. If Aurora was installed using a Debian package
generated by our `aurora-packaging` script, the recovery tool can be found
in `/usr/share/aurora/bin/recovery-tool`.

## Cleanup

* Delete (or move) the Mesos replicated log path for each scheduler instance. The location of the
Mesos replicated log file path can be found by looking at the value given to the flag
`-native_log_file_path` for each instance.

* Initialize the Mesos replicated log files using the mesos-log tool:
```
sudo -u <USER> mesos-log initialize --path=<native_log_file_path>
```
Where `USER` is the user under which the scheduler instance will be run. For installations using
Debian packages, the default user will be `aurora`. You may alternatively choose to specify
a group as well by passing the `-g <GROUP>` option to `su`.
Note that if the user under which the Aurora scheduler instance is run _does not_ have permissions
to read this directory and the files it contains, the instance will fail to start.

## Restore from backup

* Run the `recovery-tool`. Wherever the flags match those used for the scheduler instance,
use the same values:
```
$ recovery-tool -from BACKUP \
-to LOG \
-backup=<selected_backup_location> \
-native_log_zk_group_path=<native_log_zk_group_path> \
-native_log_file_path=<native_log_file_path> \
-zk_endpoints=<zk_endpoints>
```

## Bring scheduler instances back online

### If running in HA Mode

* Start the rehydrated scheduler instance along with enough cleaned up instances to
meet the `-native_log_quorum_size`. The mesos-replicated log algorithm will replenish
the "blank" scheduler instances with the information from the rehydrated instance.

* Start any remaining scheduler instances.

### If running in singleton mode

* Start the single scheduler instance.


