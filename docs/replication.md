# Replication tips

Inspired by https://blog.raveland.org/post/postgresql_lr_en/

## Master configuration: Regular Setup

The WAL level of the master needs to be set to the logical level.

First check, what the current status is:
```sql
SHOW wal_level;
```

If it is not already `logical`, and you have superuser rights, just issue the
following command:
```sql
ALTER SYSTEM SET wal_level = 'logical';
```

On the master, one needs to create a dedicated role for replication (here
`vkgreplicate`), create a publication (here `odhpub`) and grant access to the
tables to the dedicated role.
```sql
CREATE ROLE vkgreplicate WITH LOGIN PASSWORD 'Azerty' REPLICATION;
CREATE PUBLICATION odhpub FOR ALL TABLES;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO vkgreplicate;
```

## Master configuration: AWS/RDS

The WAL level of the master needs to be set to the logical level.

First check, what the current status is:
```sql
SHOW wal_level;
```

If it is not already `logical`, do: Create a new "parameter group" and within
that, set the instance parameter `rds.logical_replication` to `1` and reboot
your instance.

On the master, one needs to create a dedicated role for replication (here
`vkgreplicate`), create a publication (here `odhpub`) and grant access to the
tables to the dedicated role.
```sql
CREATE ROLE vkgreplicate WITH LOGIN PASSWORD 'Azerty';
GRANT rds_replication TO vkgreplicate;
CREATE PUBLICATION odhpub FOR ALL TABLES;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO vkgreplicate;
```

## Slave configuration

If the master is a Docker container on the same machine, one must make sure they are on the same Docker network than the master container.

Connect to the shell and open `psql`
```bash
docker exec -it my_slave_db /bin/sh
psql -U tourismuser
```

Make sure, that the slave can access the master via TCP. Check firewall rules.

Let the slave subscribe to the publication:
```sql
CREATE SUBSCRIPTION subodh CONNECTION 'host=odh-tourism-db1 dbname=tourismuser user=vkgreplicate password=Azerty' PUBLICATION odhpub;
```
Note that the subscription `subodh` must not already exist (otherwise give it another name).

### Unsubscribing

Before removing the slave container, it is recommended to disable the subscription:
```sql
ALTER SUBSCRIPTION subodh DISABLE;
```

## I see an excessive disk space usage

If you see a continues increase in storage, it is possible that PITR prevents
the cleaning of WAL logs. See [AWS RDS
Docs](https://aws.amazon.com/premiumsupport/knowledge-center/diskfull-error-rds-postgresql/).
To check whether PITR is activated or not, do
```sql
show archive_timeout;
```

If it is not equal `0`, you have PITR activated, and that means that Postgres
generates also WAL logs without writes. Normally, they would be cleaned after a
few MB, but with PITR they are kept forever. This is a problem, if you have
slots that are never consumed (`active=false`).

Find your slots and subscription with
```sql
SELECT * FROM pg_catalog.pg_subscription;
```

To free storage, you can also drop the underlying slot:
```sql
SELECT pg_drop_replication_slot('Your_slotname_name');
```