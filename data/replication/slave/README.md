# Slave Docker image for the ODH tourism dataset

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

If the master is a Docker container on the same machine, one must make sure they are on the same Docker network (here `tourism`) than the master container.

```bash
docker run --name odh_db_slave -p 7778:5432 -e POSTGRES_USER=tourismuser -e POSTGRES_PASSWORD=postgres2 --network tourism -d ontopicvkg/odh-db-slave
```

Connect to the shell and open `psql`
```bash
docker exec -it odh_db_slave /bin/sh
psql -U tourismuser
```

Make sure, that the slave can access the master via TCP. Check firewall rules.

Let the slave subscribe to the publication:
```sql
CREATE SUBSCRIPTION subodh CONNECTION 'host=odh-hackathon-2019_db_1 dbname=tourismuser user=vkgreplicate password=Azerty' PUBLICATION odhpub;
```
Note that the subscription `subodh` must not already exist (otherwise give it another name).

### Unsubscribing

Before removing the slave container, it is recommended to disable the subscription:
```sql
ALTER SUBSCRIPTION subodh DISABLE;
```