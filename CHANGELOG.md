## Version 1.3 (Unreleased)

 - use pg_checksums for "--checksums" and "--checkdb" (requires `postgresql-<version>-pg-checksums` package for PostgreSQL version 11 and below)
 - automatic determine Postgres version from pgbackrest info
 - automatic create a new postgres cluster (initdb) to restore to the path specified in the "--to" option (if it does not exist)
 - determine postgresql parameters from pg_controldata and configure postgresql.conf accordingly after restore
 - compare DB and filesystem size before restore
 - add norestore option to check already existing clusters
 - error if create extension failes
 - add an optional amcheck parameter within checkdb
 - remove colors in log messages
 - remove dependencies - gawk, ansi2html.sh
 - a little code refactoring

## Version 1.2 (20 Apr 2020)

- Bug Fixes: The pg_logical_validation() function was creating the amcheck extension, but did not actually perform indexes checking with bt_index_parent_check for PostgreSQL version 11 or later.

## Version 1.1 (27 Jan 2020)

- Compatibility with PostgreSQL versions 11 and 12.

## Version 1.0 (27 Jan 2020)

- Initial release
