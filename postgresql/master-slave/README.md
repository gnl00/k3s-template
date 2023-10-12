# postgresql-master-slave

1、创建同步账号
```postgresql
CREATE USER repl REPLICATION LOGIN ENCRYPTED PASSWORD 'repl123';
```

2、修改 `/var/lib/postgresql/data/pg_hba.conf`
```text
# TYPE DATABASE USER ADDRESS METHOD
host all repl 0.0.0.0/0 md5
```

3、修改 `/var/lib/postgresql/data/postgresql.conf`
```text
listen_addresses = '*'
wal_level = replica
archive_mode = off
max_wal_senders = 10
wal_sender_timeout = 60s
wal_keep_segments = 64
hot_standby = on
```

4、从节点同步主节点数据