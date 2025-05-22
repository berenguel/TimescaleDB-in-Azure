# **Setting Up Logical Replication with TimescaleDB**

This guide outlines the steps to set up logical replication from a source Azure Database for PostgreSQL Flexible Server to a target Azure Database for PostgreSQL Flexible Server, with TimescaleDB enabled.

## Pre Requisites

### Step 0 - Role Privileges (Target Server)

```sql
ALTER ROLE demo WITH REPLICATION;
GRANT azure_pg_admin TO demo;
```


### Step 1 - Enable TimescaleDB Extension (Target Server)

```sql
CREATE EXTENSION TIMESCALEDBD;
```

### Step 2 - Check Prerequisites for Logical Replication (Source & Target PostgreSQL Flexible Server)
Enable the below server parameters (in the server configuration)
```sql
wal_level=logical
max_worker_processes=16
max_replication_slots=10
max_wal_senders=10
track_commit_timestamp=on
```

### Step 3 - Check that all tables are ready for logical replication (source)

Tables *must have* one of the following:
- a Primary Key, or
- a unique index, or
- Replica identity full

## Let's start the Logical Replication & Data Copy

### Step 4 - Set Up Logical Replication (Source Server)

```sql
create publication logical_mig01;

alter publication logical_mig01 add table conditions;

SELECT pg_create_logical_replication_slot('logical_mig01', 'pgoutput');
```

### Step 5 - Generate Dump of Schema + Data (using Azure VM or Local Machine)

```
pg_dump -U demo -W \
-h pg14timescaledb.postgres.database.azure.com -p 5432 -Fc -v \
-f dump.bak postgres \
	-N pg_catalog \
	-N cron \
	-N information_schema
```


### Step 6 - Prepare the Target Database for TimescaleDB Copy (Target Server)

```sql
SELECT timescaledb_pre_restore();
```

### Step 7 - Restore into the Target (using Azure VM or Local Machine)
```
pg_restore -U demo -W  \
-h pg16timescale.postgres.database.azure.com -p 5432 --no-owner \
-Fc -v -d postgres dump.bak --no-acl
```

### Step 8 - Run the Post-Restore Timescale DB Command (Target Server)

```sql
SELECT timescaledb_post_restore();
```

### Step 9 - Create the Subscription on the Target & Advance Replication Origin (Target Server)

```sql
-- On the target
CREATE SUBSCRIPTION logical_sub01 CONNECTION 'host=xxxxxxx.postgres.database.azure.com port=5432 dbname=postgres user=yyyy password=zzzzzzz PUBLICATION logical_mig01
WITH (
	copy_data = false,
	create_slot = false,
	enabled = false,
	slot_name = 'logical_mig01'
);

SELECT roident, roname FROM pg_replication_origin;
-- Example output:
--  roident |  roname
-- ---------+----------
--        1 | pg_25910

-- On the source:
SELECT slot_name, restart_lsn FROM pg_replication_slots WHERE slot_name = 'logical_mig01';
-- Example output:
--  slot_name    | restart_lsn
-- -------------+-------------
--  logical_mig01 | 20/9300A690

-- On the target (replace 'pg_25910' and '20/9300A690' with your actual values):
SELECT pg_replication_origin_advance('pg_25910', '20/9300A690');
```

Step 10 - Enable the Subscription on the Target (Target Server)
```sql
ALTER SUBSCRIPTION logical_sub01 ENABLE;
```
## Let's test that the replication
Step 11 - Test Replication (Source Server)
```sql
insert into conditions values ('2025-05-21 07:00:00', 'London', 'Test', 66, 77);
select * from conditions;
```

## Let's test that the new target 
Step 12 - Test the new database  (Target Server)
```sql
insert into conditions values ('2025-05-22 07:00:00', 'Edinburgh', 'Test3', 66, 77);
select * from conditions;
```
```
