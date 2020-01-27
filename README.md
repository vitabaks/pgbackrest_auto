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
pgbackrest_auto --from=app-db --to=/bkpdata/10/app-db --backup-host=10.128.50.50 --pgver=10 --checkdb --clear --report
```
###### script output:
```
2019-06-17 15:57:50 INFO: [STEP 1]: Starting
2019-06-17 15:57:50 INFO: Starting. Restore Type: Full PostgreSQL Restore FROM Stanza: app-db ---> TO Directory: /bkpdata/10/app-db
2019-06-17 15:57:50 INFO: Starting. Restore Settings: immediate
2019-06-17 15:57:50 INFO: Starting. Run settings: Backup host: 10.128.50.50
2019-06-17 15:57:50 INFO: Starting. Run settings: Log: /var/log/pgbackrest/pgbackrest_auto-restore_app-db.log
2019-06-17 15:57:50 INFO: Starting. Run settings: Lock run: /tmp/pgbackrest_auto_app-db.lock
2019-06-17 15:57:50 INFO: Starting. PostgreSQL instance: app-db
2019-06-17 15:57:50 INFO: Starting. PostgreSQL version: 10
2019-06-17 15:57:50 INFO: Starting. PostgreSQL port: 7433
2019-06-17 15:57:50 INFO: Starting. PostgreSQL Database Validation: yes
2019-06-17 15:57:50 INFO: Starting. Clear Data Directory after restore: yes
2019-06-17 15:57:50 WARN: Restoring to /bkpdata/10/app-db Waiting 30 seconds. The directory will be overwritten. If mistake, press ^C
2019-06-17 15:58:20 INFO: [STEP 2]: Stopping PostgreSQL
2019-06-17 15:58:20 INFO: attempt: 1/3600
2019-06-17 15:58:20 INFO: PostgreSQL check status
2019-06-17 15:58:20 INFO: /bkpdata/10/app-db is not a database cluster directory. May be its clean.
2019-06-17 15:58:20 INFO: [STEP 3]: Restoring from backup
2019-06-17 15:58:20 INFO: Restore from backup started. Type: Full PostgreSQL Restore
2019-06-17 15:58:20 INFO: See detailed log in the file /var/log/pgbackrest/app-db-restore.log
2019-06-17 16:09:56 INFO: Restore from backup done
2019-06-17 16:09:56 INFO: [STEP 4]: PostgreSQL Starting for recovery
2019-06-17 16:09:56 INFO: PostgreSQL start
2019-06-17 16:10:02 INFO: attempt: 1/3600
2019-06-17 16:10:03 INFO: PostgreSQL instance app-db started and accepting connections
2019-06-17 16:10:03 INFO: [STEP 5]: PostgreSQL Recovery Checking
2019-06-17 16:10:03 INFO: Checking if restoring from archive is done
2019-06-17 16:10:03 INFO: Replayed: 2019-06-17 02:14:25.695946+03
2019-06-17 16:10:03 INFO: Restoring from archive is done
2019-06-17 16:10:03 INFO: Restore done
2019-06-17 16:10:03 INFO: [STEP 6]: Validate for physical database corruption
2019-06-17 16:10:03 INFO: Start data validation for database db1
2019-06-17 16:12:22 INFO: Data validation in the database db1 - Successful
2019-06-17 16:12:22 INFO: Start data validation for database db2
2019-06-17 16:12:22 INFO: Data validation in the database db2 - Successful
2019-06-17 16:12:22 INFO: Start data validation for database db3
2019-06-17 16:12:28 INFO: Data validation in the database db3 - Successful
2019-06-17 16:14:08 INFO: Start data validation for database postgres
2019-06-17 16:14:09 INFO: Data validation in the database postgres - Successful
2019-06-17 16:14:09 INFO: [STEP 7]: Validate for logical database corruption (with amcheck)
2019-06-17 16:14:09 INFO: Verify the logical consistency of the structure of indexes and heap relations in the database db1
2019-06-17 16:17:44 INFO: Verify the logical consistency of the structure of indexes and heap relations in the database db2
2019-06-17 16:17:44 INFO: Verify the logical consistency of the structure of indexes and heap relations in the database db3
2019-06-17 16:20:03 INFO: Verify the logical consistency of the structure of indexes and heap relations in the database postgres
2019-06-17 16:20:04 INFO: [STEP 8]: Stopping PostgreSQL and Clear Data Directory
2019-06-17 16:20:08 INFO: PostgreSQL stop
2019-06-17 16:20:08 INFO: PostgreSQL instance app-db stopped
2019-06-17 16:20:08 INFO: attempt: 1/3600
2019-06-17 16:20:08 INFO: PostgreSQL check status
2019-06-17 16:20:08 INFO: PostgreSQL instance app-db not running
2019-06-17 16:20:09 INFO: Directory /bkpdata/10/app-db is cleared
2019-06-17 16:20:09 INFO: [STEP 9]: Send report to mail address
```


#### pgbackrest_auto --help
```
/usr/bin/pgbackrest_auto

Automatic Restore PostgreSQL from backup

Support three types of restore:
        1) Restore last backup  (recovery to earliest consistent point) [default]
        2) Restore latest       (recovery to the end of the archive stream)
        3) Restore to the point (recovery to restore point)

Important: Run on the nodes on which you want to restore the backup

Usage: /usr/bin/pgbackrest_auto --from=STANZANAME --to=DATA_DIRECTORY [ --datname=DATABASE [...] ] [ --recovery-type=( default | immediate | time ) ] [ --recovery-target=TIMELINE  [ --backup-set=SET ] [ --backup-host=HOST ] [ --pgver=( 94 | 10 ) ] [ --checkdb ] [ --clear ] [ --report ] ]


--from=STANZANAME
        Stanza from which you need to restore from a backup

--to=DATA_DIRECTORY
        PostgreSQL Data directory Path to restore from a backup
        Example: /var/lib/postgresql/11/rst

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
        Example: "2018-08-08 12:46:54"

--backup-set=SET
        If you need to restore not the most recent backup. Example few days ago.
        Get info of backup. Login to pgbackrest server. User postgres
        pgbackrest --stanza=[STANZA NAME] info
        And get it. Example:
                    incr backup: 20180807-212125F_20180808-050002I
        This is the name of SET: 20180807-212125F_20180808-050002I

--backup-host=HOST
        pgBacRest repository ip address (Use SSH Key-Based Authentication)
        localhost [default]

--pgver=VERSION
        PostgreSQL cluster (instance) version
        94 | 95 | 96 | 10 [default] | 11 | 12

--checkdb
        Validate for Physical and Logical Database Corruption (It work with only Full PostgreSQL Restore)

--clear
        Clear PostgreSQL Data directory after Restore (the path was specified in the "--to" parameter ) [ optional ]

--report
        Send report to mail address (Specify smtp parameters in the /usr/bin/pgbackrest_auto file)



EXAMPLES:
( example stanza "app-db" , backup-host "localhost" )

| Restore last backup.

    # /usr/bin/pgbackrest_auto --from=app-db --to=/var/lib/postgresql/11/rst

| Restore latest backup (recover to the end of the archive stream).

    # /usr/bin/pgbackrest_auto --from=app-db --to=/var/lib/postgresql/11/rst --recovery-type=default

| Restore backup made a few days ago.

    # /usr/bin/pgbackrest_auto --from=app-db --to=/var/lib/postgresql/11/rst --backup-set=20180807-212125F_20180808-050002I

| Restore backup made a few days ago and pick time.

    # /usr/bin/pgbackrest_auto --from=app-db --to=/var/lib/postgresql/11/rst --backup-set=20180807-212125F_20180808-050002I --recovery-type=time --recovery-target="2018-08-08 12:46:54"

| Restore backup made a few days ago and pick time. And we have restore only one database with the name "app_db".

    # /usr/bin/pgbackrest_auto --from=app-db --to=/var/lib/postgresql/11/rst --datname=app_db --backup-set=20180807-212125F_20180808-050002I --recovery-type=time --recovery-target="2018-08-08 12:46:54"

| Restore and Validate of databases (for example: pgBacRest repository 10.128.64.50, PostgreSQL version 11)

    # /usr/bin/pgbackrest_auto --from=app-db --to=/var/lib/postgresql/11/rst --backup-host=10.128.64.50 --pgver=11 --checkdb

```

---

 :bulb: You can use this script to daily (or weekly) automatically check your backups, immediately after the completion of the backup process.

###### Example of Cron jobs:

```
#=== pgbackrest - Backup PostgreSQL ====================

01 00 * * 6 if pgbackrest --stanza=app-db --type=full backup; then pgbackrest_auto --from=app-db --to=/bkpdata/10/app-db --backup-host=10.128.50.50 --checkdb --clear --report; fi
01 00 * * 0-5 if pgbackrest --stanza=app-db --type=diff backup; then pgbackrest_auto --from=app-db --to=/bkpdata/10/app-db --backup-host=10.128.50.50 --checkdb --clear --report; fi

30 00 * * 6 if pgbackrest --stanza=apdb-cluster --type=full backup; then pgbackrest_auto --from=apdb-cluster --to=/bkpdata/11/apdb-cluster --backup-host=10.128.50.50 --pgver=11 --checkdb --clear --report; fi
30 00 * * 0-5 if pgbackrest --stanza=apdb-cluster --type=diff backup; then pgbackrest_auto --from=apdb-cluster --to=/bkpdata/11/apdb-cluster --backup-host=10.128.50.50 --pgver=11 --checkdb --clear --report; fi

00 01 * * 6 if pgbackrest --stanza=dbs-eu--type=full backup; then pgbackrest_auto --from=dbs-eu--to=/bkpdata/9.4/dbs-eu--backup-host=10.128.50.50 --pgver=94 --checkdb --clear --report; fi
00 01 * * 0-5 if pgbackrest --stanza=dbs-eu--type=diff backup; then pgbackrest_auto --from=dbs-eu--to=/bkpdata/9.4/dbs-eu--backup-host=10.128.50.50 --pgver=94 --checkdb --clear --report; fi

#=======================================================
```

> *I recommend to pre-init PGDATA for each stanza (PostgreSQL clusters)*

```
$ pg_lsclusters
Ver Cluster     Port Status Owner    Data directory                  Log file
9.4 dbs-eu      7435 down   postgres /bkpdata/9.4/dbs-eu       /var/log/postgresql/postgresql-9.4-app-eu.log
10 app-db       7433 down   postgres /bkpdata/10/app-db        /var/log/postgresql/postgresql-10-app-db.log
11 apdb-cluster 7434 down   postgres /bkpdata/9.4/apdb-cluster /var/log/postgresql/postgresql-11-apdb-cluster.log
...
```

## Compatibility
Debian/Ubuntu

:white_check_mark: tested, works fine: `Debian 8/9`

###### PostgreSQL versions: 
all supported PostgreSQL versions

:white_check_mark: tested, works fine: `PostgreSQL 9.4, 10, 11`

## Requirements
pgbackrest >= 2.01

for `--checkdb`:
- [amcheck_next](https://github.com/petergeoghegan/amcheck) extension/SQL version >=2 `(if PostgreSQL version <= 10)`.

>You can use the packages for your PostgreSQL version from the [APT](https://apt.postgresql.org/pub/repos/apt/pool/main/a/amcheck/) repository 

for `--report`: 
- sendemail
- gawk
- [ansi2html.sh](https://github.com/pixelb/scripts/blob/master/scripts/ansi2html.sh) script
- Specify smtp parameters `smtp_server`, `mail_from`, `mail_to` in the `/usr/bin/pgbackrest_auto` file

local `trust` for `postgres` (login by Unix domain socket) in the `pg_hba.conf` or use `.pgpass` file.

Pre-init `initdb` PGDATA for your stanza (PostgreSQL cluster/instance)

Run as user: `postgres`


## Installation
1. Download and copy the `pgbackrest_auto` script to `/usr/bin/` directory
2. Download and copy the `ansi2html.sh` script to `/usr/bin/` directory
3. Grant execute rights on the scripts
4. Install `amcheck` package into your system 
> the amcheck extension will be automatically installed to the restored databases


## License
Licensed under the MIT License. See the [LICENSE](./LICENSE) file for details.

## Author
Vitaliy Kukharik (PostgreSQL DBA) vitabaks@gmail.com

## Feedback, bug-reports, requests, ...
Are [welcome](https://github.com/vitabaks/pgbackrest_auto/issues)!
#### Help Wanted! If you noticed a bug or a missing feature or just have an idea of how this project could be enhanced, please feel free to file an issue.
