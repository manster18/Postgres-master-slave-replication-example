# Postgres Master Slave Failover & Failback

![image](https://img.youtube.com/vi/lnY9MIyiALY/0.jpg)

# start

```bash
docker run -it --rm -p 5551:5432 --name=db1 --hostname=db1 ubuntu:20.04 bash
docker run -it --rm -p 5552:5432 --name=db2 --hostname=db2 ubuntu:20.04 bash
```

# on both containers

```bash
ln -snf /usr/share/zoneinfo/UTC /etc/localtime && echo UTC > /etc/timezone
apt update
apt install -y postgresql postgresql-contrib iputils-ping sudo vim netcat

echo "listen_addresses = '*'" >> /etc/postgresql/12/main/postgresql.conf
echo "wal_log_hints = on" >> /etc/postgresql/12/main/postgresql.conf

echo "archive_mode = on" >>  /etc/postgresql/12/main/postgresql.conf
echo "archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'" >>  /etc/postgresql/12/main/postgresql.conf
# echo "archive_cleanup_command = 'pg_archivecleanup /archive %r'" >>  /etc/postgresql/12/main/postgresql.conf
# echo "restore_command = 'cp /archive/%f %p'" >>  /etc/postgresql/12/main/postgresql.conf


echo "host replication replica 0.0.0.0/0 md5" >> /etc/postgresql/12/main/pg_hba.conf
echo "host all rewind 0.0.0.0/0 md5" >> /etc/postgresql/12/main/pg_hba.conf


mkdir /archive
chown -R postgres:postgres /archive
```

# on host machine

```bash
docker inspect db1 | grep IPAddress # db1 - 172.17.0.2
docker inspect db2 | grep IPAddress # db2 - 172.17.0.3
```

# on first container

first of all we gonna need to recreate cluster with checksums enabled

check connectivity by sending pings

```bash
ping 172.17.0.3
```

start postgres

```bash
pg_ctlcluster 12 main start
```

create `replica` and `rewind` users with password `123`

```bash
sudo -u postgres psql -c "create user replica with replication encrypted password '123'"
sudo -u postgres psql -c "CREATE USER rewind SUPERUSER encrypted PASSWORD '123'"
```

create `sample` database and fill it with approx 500mb of data

```bash
sudo -u postgres psql -c "create database sample"
sudo -u postgres pgbench -i -s 10 sample
```

# on a second container

cleanup data directory

```bash
sudo -u postgres rm -rf /var/lib/postgresql/12/main/*
```

ensure first container is listening on expected port

```bash
nc -vz 172.17.0.2 5432
```

restore cluster from master (it will ask for `123` password of `replica` user, also note that it can take some time to backup restore 500mb)

```bash
sudo -u postgres pg_basebackup --host=172.17.0.2 --port=5432 --username=replica --pgdata=/var/lib/postgresql/12/main/ --progress --write-recovery-conf --create-slot --slot=replica1
```

notes:

- it will ask for `123` password of `replica` user created earlier
- it might take some time to backup restore 500mb of data
- it will wait for a checkpoint before starting, so run `sudo -u postgres psql -c "checkpoint"` on a master


make sure that connection info is saved

```bash
cat /var/lib/postgresql/12/main/postgresql.auto.conf
```

and that you have `standby.signal` file in place (existence of this file will force postgres to run as slave)

```bash
ls -la /var/lib/postgresql/12/main/ | grep standby
```

start postgres

```bash
pg_ctlcluster 12 main start
```

and make sure it is up and running

```bash
pg_lsclusters
```

you should see `online,recovery`

# on a first container

check replication slots

```bash
sudo -u postgres psql -c "select * from pg_replication_slots"
sudo -u postgres psql -c "select * from pg_stat_replication"
```

lets create table and fill it with some dummy data

```bash
sudo -u postgres psql sample -c "create table messages(message text)"
sudo -u postgres psql sample -c "insert into messages values('hello')"
sudo -u postgres psql sample -c "select * from messages"
```

almost immediatelly you should see that table and message on a replica

# Failover

## on a second container

lets pretend that we lose our master

promote second container as a new master

```bash
sudo pg_ctlcluster 12 main promote
```

standby file should be removed automatically

```bash
ls -la /var/lib/postgresql/12/main/ | grep standby
```

> Connection info in `postgres.auto.conf` will left inact, but it is ok, until there is no standby file

you should be able to write records now

```bash
sudo -u postgres psql sample -c "insert into messages values('world')"
sudo -u postgres psql sample -c "select * from messages"
```

## on a first container

but master is still alive and received one more update

```bash
sudo -u postgres psql sample -c "insert into messages values('contoso')"
sudo -u postgres psql sample -c "select * from messages"
```

stop postgres

```bash
pg_ctlcluster 12 main stop
```

rewind

```bash
sudo -u postgres /usr/lib/postgresql/12/bin/pg_rewind --target-pgdata /var/lib/postgresql/12/main --source-server="postgresql://rewind:123@172.17.0.3:5432/sample" --progress
```

rewind might complain with error like: `pg_rewind: error: could not open file "/var/lib/postgresql/12/main/pg_wal/00000001000000000000000A": No such file or directory` you gonna need to copy this file from `/archive` to `pg_wal`, e.g.:

```bash
cp /archive/00000001000000000000000A /var/lib/postgresql/12/main/pg_wal/
chown postgres:postgres /var/lib/postgresql/12/main/pg_wal/00000001000000000000000A
```

note: in pg13 rewind can add connection info

now, when we rewinded lets make it slave

```bash
touch /var/lib/postgresql/12/main/standby.signal
```

and add replication info

```bash
echo "primary_conninfo = 'user=replica password=123 host=172.17.0.3 port=5432 sslmode=prefer sslcompression=0 gssencmode=prefer krbsrvname=postgres target_session_attrs=any'" >> /var/lib/postgresql/12/main/postgresql.auto.conf
echo "primary_slot_name = 'replica2'" >> /var/lib/postgresql/12/main/postgresql.auto.conf
```

note: do not forget to change address and replication slot name


TODO: create slot on new master

```bash
sudo -u postgres psql -c "select * from pg_create_physical_replication_slot('replica2')"
sudo -u postgres psql -c "select * from pg_replication_slots"
```

start postgres

```bash
pg_ctlcluster 12 main start
```

and make sure it is up and running

```bash
pg_lsclusters
```

check that data is synced

```bash
sudo -u postgres psql sample -c "select * from messages"
```

important: just in case check that old `replica1` is not here and removed

```bash
sudo -u postgres psql -c "select * from pg_replication_slots"
sudo -u postgres psql -c "select * from pg_drop_replication_slot('replica1')"
```
