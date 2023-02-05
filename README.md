# pgbackrest_auto

This is a simple script to automate the process of restore PostgreSQL databases from backup using [pgbackrest](https://github.com/pgbackrest/pgbackrest). With the possibility **validate for physical and logical database corruption**, and (optional) sending the final report to DBA an e-mail address.

#### Data Validation

>PostgreSQL has supported page-level checksums since 9.3. **If page checksums are enabled** pgbackrest will [validate](https://pgbackrest.org/configuration.html#section-backup/option-checksum-page) the checksums for every file that is copied during a backup. All page checksums are validated during a full backup and checksums in files that have changed are validated during differential and incremental backups.
This feature allows page-level corruption to be detected early.

`pgbackrest_auto` runs **additional advanced checks on your data** after the recovery stage. In case of PostgreSQL data restore and recovery, the stage of physical data validation is performed (all data blocks are sequentially read with pg_dump);
Databases that successfully passed the stage of physical validation additionally pass the stage of logical validation using the [amcheck](https://github.com/petergeoghegan/amcheck) extension. Tthe logical integrity of the structure of the B-Tree indexes and tables related to the target index relation is checked (used `bt_index_parent_check` with `heapallindexed` argument is `true`).

![pgbackrest_auto_scheme](./scheme.png)

#### Example
```
pgbackrest_auto --from=epgdb --to=/bkpdata/rst/epgdb --checkdb
```
###### result:
```
2022-06-15 14:13:13 INFO: [STEP 1]: Starting
2022-06-15 14:13:13 INFO: Starting. Restore Type: Full PostgreSQL Restore FROM Stanza: epgdb --> TO Directory: /bkpdata/rst/epgdb
2022-06-15 14:13:14 INFO: Starting. Restore Settings: immediate
2022-06-15 14:13:14 INFO: Starting. Run settings: Backup host: localhost
2022-06-15 14:13:14 INFO: Starting. Run settings: Log: /var/log/pgbackrest/pgbackrest_auto_epgdb.log
2022-06-15 14:13:14 INFO: Starting. Run settings: Lock run: /tmp/pgbackrest_auto_epgdb.lock
2022-06-15 14:13:14 INFO: Starting. PostgreSQL instance: epgdb
2022-06-15 14:13:14 INFO: Starting. PostgreSQL version: 11
2022-06-15 14:13:14 INFO: Starting. PostgreSQL port: 5432
2022-06-15 14:13:14 INFO: Starting. PostgreSQL Database Validation: yes
2022-06-15 14:13:14 WARN: Restoring to /bkpdata/rst/epgdb Waiting 30 seconds. The directory will be overwritten. If mistake, press ^C
2022-06-15 14:13:44 INFO: [STEP 2]: Stopping PostgreSQL
2022-06-15 14:13:44 INFO: attempt: 1/3600
2022-06-15 14:13:44 INFO: PostgreSQL check status
2022-06-15 14:13:44 INFO: PostgreSQL instance epgdb not running
2022-06-15 14:13:44 INFO: [STEP 3]: Restoring from backup
2022-06-15 14:13:44 INFO: Restore from backup started. Type: Full PostgreSQL Restore
2022-06-15 14:13:44 INFO: See detailed log in the file /var/log/pgbackrest/epgdb-restore.log
pgbackrest --config=/tmp/pgbackrest_auto.conf --repo1-host=localhost --repo1-host-user=postgres --stanza=epgdb --pg1-path=/bkpdata/rst/epgdb  --type=immediate --repo1-path=/bkpdata/pgbackrest --delta restore --process-max=4 --log-level-console=error --log-level-file=detail --recovery-option=recovery_target_action=promote --tablespace-map-all=/bkpdata/rst/epgdb/remapped_tablespaces
2022-06-15 14:13:52 INFO: Restore from backup done
2022-06-15 14:13:52 INFO: [STEP 4]: PostgreSQL Starting for recovery
2022-06-15 14:13:52 INFO: PostgreSQL start
2022-06-15 14:14:02 INFO: attempt: 1/3600
2022-06-15 14:14:03 INFO: PostgreSQL instance epgdb started and accepting connections
2022-06-15 14:14:03 INFO: [STEP 5]: PostgreSQL Recovery Checking
2022-06-15 14:14:03 INFO: Checking if restoring from archive is done
2022-06-15 14:14:03 INFO: Replayed:
2022-06-15 14:14:03 INFO: Restoring from archive is done
2022-06-15 14:14:03 INFO: Restore done
2022-06-15 14:14:03 INFO: [STEP 6]: Validate for physical database corruption
2022-06-15 14:14:04 INFO: Start data validation for database postgres
2022-06-15 14:14:04 INFO: ... starting pg_dump -p 5432 -d postgres >> /dev/null
2022-06-15 14:14:06 INFO: Data validation in the database postgres - Successful
2022-06-15 14:14:06 INFO: Start data validation for database epg
2022-06-15 14:14:06 INFO: ... starting pg_dump -p 5432 -d epg >> /dev/null
2022-06-15 14:14:12 INFO: Data validation in the database epg - Successful
2022-06-15 14:14:12 INFO: Start data validation for database ota
2022-06-15 14:14:12 INFO: ... starting pg_dump -p 5432 -d ota >> /dev/null
2022-06-15 14:14:13 INFO: Data validation in the database ota - Successful
2022-06-15 14:14:13 INFO: [STEP 7]: Validate for logical database corruption
2022-06-15 14:14:13 INFO: pg_checksums: starting data checksums validation
2022-06-15 14:14:21 INFO: pg_checksums: data checksums validation - Successful
2022-06-15 14:14:22 INFO: amcheck: verify the logical consistency of the structure of indexes and heap relations in the database postgres
2022-06-15 14:14:22 INFO: amcheck: verify the logical consistency of the structure of indexes and heap relations in the database epg
2022-06-15 14:14:59 INFO: amcheck: verify the logical consistency of the structure of indexes and heap relations in the database ota
2022-06-15 14:15:23 INFO: Finish
```


#### pgbackrest_auto --help
```
Automatic Restore and Validate for physical and logical database corruption

Support three types of restore:
        1) Restore last backup  (recovery to earliest consistent point) [default]
        2) Restore latest       (recovery to the end of the archive stream)
        3) Restore to the point (recovery to restore point)

Important: Run on the nodes on which you want to restore the backup

Usage: /usr/bin/pgbackrest_auto --from=STANZANAME --to=DATA_DIRECTORY [ --datname=DATABASE [...] ] [ --recovery-type=( default | immediate | time ) ] [ --recovery-target=TIMELINE  [ --backup-set=SET ] [ --backup-host=HOST ] [ --pgver= ] [ --checkdb ] [ --clear ] [ --report ] ]

--from=STANZANAME
        Stanza from which you need to restore from a backup

--to=DATA_DIRECTORY
        PostgreSQL Data directory Path to restore from a backup
        a PostgreSQL database cluster (PGDATA) will be automatically created (initdb) if it does not exist
        Example: /bkpdata/rst/app-db

--datname=DATABASE [...]
        Database name to be restored (After this you MUST drop other databases)
        Note that built-in databases (template0, template1, and postgres) are always restored.
        To be restore more than one database specify them in brackets separated by spaces.
        Example: --datname="db1 db2"

--recovery-type=TYPE
        immediate - recover only until the database becomes consistent           (Type 1. Restore last backup)  [default]
        default   - recover to the end of the archive stream                     (Type 2. Restore latest)
        time      - recover to the time specified in --recovery-target           (Type 3. Restore to the point)

--recovery-target=TIMELINE
        time - recovery point time. The time stamp up to which recovery will proceed.
        if --recovery-type=time
        Example: "2022-06-14 09:00:00"

--backup-set=SET
        If you need to restore not the most recent backup. Example few days ago.
        Get info of backup. Login to pgbackrest server. User postgres
        pgbackrest --stanza=[STANZA NAME] info
        And get it. Example:
                    incr backup: 20220611-000004F_20220614-000003D
        This is the name of SET: 20220611-000004F_20220614-000003D

--backup-host=HOST
        pgBacRest repository ip address (Use SSH Key-Based Authentication)
        localhost [default]

--pgver=VERSION
        PostgreSQL cluster (instance) version [ optional ]
        by default, the PostgreSQL version will be determined from the pgbackrest info

--dummy-dump
        Verify that data can be read out. Check with pg_dump >> /dev/null

--checksums
        Check data checksums

--amcheck
        Validate Indexes (verify the logical consistency of the structure of indexes and heap relations)

--checkdb
        Validate for Physical and Logical Database Corruption (includes: dummy-dump, checksums, amcheck)

--clear
        Clear PostgreSQL Data directory after Restore (the path was specified in the "--to" parameter ) [ optional ]

--report
        Send report to mail address

--norestore
        Do not restore a stanza but use an already existing cluster

--config=/path/to/pgbackrest.conf
        The path to the custom pgbackrest configuration file [ optional ]

--custom-options=""
        Costom options for pgBackRest [ optional ]
	This includes all the options that may also be configured in pgbackrest.conf
        Example: "--option1=value --option2=value --option3=value"
        See all available options: https://pgbackrest.org/configuration.html


EXAMPLES:
( example stanza "app-db" , backup-host "localhost" )

| Restore last backup:

    /usr/bin/pgbackrest_auto --from=app-db --to=/bkpdata/rst/app-db

| Restore backup made a few days ago:

    /usr/bin/pgbackrest_auto --from=app-db --to=/bkpdata/rst/app-db --backup-set=20220611-000004F_20220614-000003D

| Restore backup made a few days ago and pick time:

    /usr/bin/pgbackrest_auto --from=app-db --to=/bkpdata/rst/app-db --backup-set=20220611-000004F_20220614-000003D --recovery-type=time --recovery-target="2022-06-14 09:00:00"

| Restore backup made a few days ago and pick time. And we have restore only one database with the name "app_db":

    /usr/bin/pgbackrest_auto --from=app-db --to=/bkpdata/rst/app-db --backup-set=20220611-000004F_20220614-000003D --recovery-type=time --recovery-target="2022-06-14 09:00:00" --datname=app_db

| Restore and Validate of databases:

    /usr/bin/pgbackrest_auto --from=app-db --to=/bkpdata/rst/app-db --checkdb

```

---

 :bulb: You can use this script to daily automatically check your backups, immediately after the completion of the backup process.

###### Example of Cron jobs:

```
#=== pgbackrest - Backup PostgreSQL ====================

01 00 * * 6 if pgbackrest --stanza=app-db --type=full backup; then pgbackrest_auto --from=app-db --to=/bkpdata/rst/app-db --checkdb --clear --report; fi
01 00 * * 0-5 if pgbackrest --stanza=app-db --type=diff backup; then pgbackrest_auto --from=app-db --to=/bkpdata/rst/app-db --checkdb --clear --report; fi

30 00 * * 6 if pgbackrest --stanza=apdb-cluster --type=full backup; then pgbackrest_auto --from=apdb-cluster --to=/bkpdata/rst/apdb-cluster --checkdb --clear --report; fi
30 00 * * 0-5 if pgbackrest --stanza=apdb-cluster --type=diff backup; then pgbackrest_auto --from=apdb-cluster --to=/bkpdata/rst/apdb-cluster --checkdb --clear --report; fi

00 01 * * 6 if pgbackrest --stanza=dbs-eu --type=full backup; then pgbackrest_auto --from=dbs-eu--to=/bkpdata/rst/dbs-eu --checkdb --clear --report; fi
00 01 * * 0-5 if pgbackrest --stanza=dbs-eu --type=diff backup; then pgbackrest_auto --from=dbs-eu--to=/bkpdata/rst/dbs-eu --checkdb --clear --report; fi

#=======================================================
```

## Compatibility
RedHat and Debian based distros

###### PostgreSQL versions:
all supported PostgreSQL versions

## Requirements
`pgbackrest` and `jq` packages.

for `--checksums` (and `--checkdb`):
  - `postgresql-<version>-pg-checksums` package (if PostgreSQL version <= 11)

for `--amcheck` (and `--checkdb`):
  - `postgresql-<version>-amcheck` package (if PostgreSQL version <= 10)
    (_the amcheck extension will be automatically installed to the restored databases_)

for `--report`:
  - `sendemail` package
  - specify smtp parameters `smtp_server`, `mail_from`, `mail_to` in the pgbackrest_auto script file.

Run as user: `postgres`

If your PostgreSQL is installed somewhere other than the default installation path, please specify the `PG_BIN_DIR` variable in the script file.

## Installation
1. Download and copy the `pgbackrest_auto` script to `/usr/bin/` directory
2. Grant execute rights on the scripts

Example:
```
wget https://raw.githubusercontent.com/vitabaks/pgbackrest_auto/master/pgbackrest_auto
sudo mv pgbackrest_auto /usr/bin/
sudo chown postgres:postgres /usr/bin/pgbackrest_auto
sudo chmod 750 /usr/bin/pgbackrest_auto
```


## Logging
Log file: `/var/log/pgbackrest/pgbackrest_auto_<STANZANAME>.log`

In addition, the script execution is written in syslog. Get the pgbackrest_auto log:
```
sudo grep pgbackrest_auto /var/log/syslog
```

## License
Licensed under the MIT License. See the [LICENSE](./LICENSE) file for details.

## Author
Vitaliy Kukharik (PostgreSQL DBA) vitabaks@gmail.com

## Feedback, bug-reports, requests, ...
Are [welcome](https://github.com/vitabaks/pgbackrest_auto/issues)!
#### Help Wanted! If you noticed a bug or a missing feature or just have an idea of how this project could be enhanced, please feel free to file an issue.
