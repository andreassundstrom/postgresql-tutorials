# Restore from wal-archive

This guide will demonstrate how to set up wal-archiving and restore a backup from the archives.

### Start the postgres cluster

In a terminal run `docker compose up -d` to start the postgres cluster.

The following folders are mounted to the local filesystem:

```yaml
volumes:
  - ./pgdata/data:/var/lib/postgresql/data
  - ./pgdata/backup:/var/lib/postgresql/backup
  - ./pgdata/archive:/var/lib/postgresql/archive
```

In order for the postgres process to be able to archive wal-files to the archive folder run the following command:

```ps
docker exec db chown postgres:postgres /var/lib/postgresql/archive
```

### Create a base backup

Create a base backup:

```ps
docker exec db pg_basebackup -U postgres -D /var/lib/postgresql/backup
```

Inspect the folder `pgdata/backup`. It contains a backup of the postgres cluster.

### Enable WAL Archiving

Inspect the file `pgdata/data/postgresql.conf`. This is the main configuration file for the cluster. See the section named `- Archiving -`. The default value for archiving is off, turn it on and set the archive command:

```ini
# - Archiving -

archive_mode = on
archive_command = 'test ! -f /var/lib/postgresql/archive/%f && cp %p /var/lib/postgresql/archive/%f'
```

Restart the container with `docker compose restart` then stop the container with `docker compose stop`. Inspect the folder `pgdata/archive`. During shut down, the cluster will archive the active WAL files.

### Create some data

Start the cluster again with `docker compose start`. Connect to the cluster using the `psql` tool with `docker exec -it db psql -U postgres`. Run the following command to create a table:

```sql
CREATE TABLE employees (
  id INT GENERATED ALWAYS AS IDENTITY NOT NULL,
  name VARCHAR(200),
  created timestamp DEFAULT current_timestamp
);
```

Insert data into the table:

```sql
INSERT INTO employees (name) VALUES ('Kitty Joella');
```

Wait a minute and insert some more data:

```sql
INSERT INTO employees (name) VALUES ('Roman Jessamine');
```

And after yet a minute insert some more data:

```sql
INSERT INTO employees (name) VALUES ('Kaylee Haven');
```

Inspect the data by running `SELECT * FROM employees;`:

```txt
 id |      name       |          created
----+-----------------+----------------------------
  1 | Kitty Joella    | 2024-02-02 19:51:46.582116
  2 | Roman Jessamine | 2024-02-02 19:53:03.880468
  3 | Kaylee Haven    | 2024-02-02 19:54:30.435581
```

Shut the cluster down with `docker compose stop`.

### Restore the cluster

Delete all the files in the folder `pgdata/data`, then copy and paste the content from the `pgdata/backup` to the `pgdata/data` folder.

Update the file `postgresql.conf`. Uncomment and change the commands under `- Archive Recovery -` to:

```ini
restore_command = 'cp /var/lib/postgresql/archive/%f %p'
```

Remove the content of the folder `pgdata/data/pg_wal`

Finaly and empty file in `pgdata/data/recovery.signal`.

Start the cluster with `docker compose start`.

Verify that the correct data exists:

```
> docker exec db psql -U postgres -c "SELECT * FROM employees"

 id |      name       |          created
----+-----------------+----------------------------
  1 | Kitty Joella    | 2024-02-02 19:51:46.582116
  2 | Roman Jessamine | 2024-02-02 19:53:03.880468
  3 | Kaylee Haven    | 2024-02-02 19:54:30.435581
```

### Restore to a point in time

Once again stop the cluster with `docker compose stop`, delete the content of `pgdata/data` and replace it with the content from `pgdata/backup`. Create the file `pgdata/data/recover.signal`. Modify the file `pgdata/data/postgresql.conf`, this time also setting a recovery target time between the creation of the second row but before the last row:

```ini
restore_command = 'cp /var/lib/postgresql/archive/%f %p'
recovery_target_time = '2024-02-02 19:53:23.880468 UTC'
```

Start the cluster again with `docker compose start`. The logs from the container should state that recovery has stopped before the last transaction was commited:

```text
LOG:  recovery stopping before commit of transaction 745, time 2024-02-02 19:54:30.435773+00
```

Querying the database should also show that only the first two rows are restored:

```text
> docker exec db psql -U postgres -c "SELECT * FROM employees

 id |      name       |          created
----+-----------------+----------------------------
  1 | Kitty Joella    | 2024-02-02 19:51:46.582116
  2 | Roman Jessamine | 2024-02-02 19:53:03.880468
```
