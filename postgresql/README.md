# postgresql

## 安装

1、[Ubuntu install](https://www.postgresql.org/download/linux/ubuntu/)

2、`psql -V` 查看版本

3、切换用户，并登录
```shell
sudo -u postgres psql
```

* 配置路径：`/etc/postgresql/16/main/`
* 数据路径：`/var/lib/postgresql/16/main/`
* 配置模板路径：`/usr/share/postgresql/16/main/`
* pg 命令行工具：`/usr/lib/postgresql/16/bin`

> 可能会遇到 `/var/lib/postgresql/` 目录下 *operation not permitted* 的问题，解决方法：
> * 一次性修改：`chown -R postgres:postgres /var/lib/postgresql/`
> * 先修改 owner：`chown -R postgres /var/lib/postgresql/`，再修改 group：`chown -R postgres /var/lib/postgresql/`

4、创建用户
```postgresql
CREATE USER root superuser login PASSWORD '123456';
```

5、开启远程访问

5.1、修改 pg_hba.conf
```text
# TYPE DATABASE USER ADDRESS METHOD
host all all 0.0.0.0/0 md5
```

5.2、修改 postgresql.conf
```text
listen_addresses = '*'
```

5.3、重启服务
```shell
sudo service postgresql restart
# or
sudo /etc/init.d/postgresql restart
# or
sudo systemctl restart postgresql
```

> 现在就可以通过远程访问数据库了

6、创建测试数据
```postgresql
CREATE DATABASE test OWNER root;

create table myuser (
    id serial primary key,
    name varchar(20) not null,
    age int not null
);

insert into myuser (name, age) values ('zhangsan', 20);
insert into myuser (name, age) values ('lisi', 21);
insert into myuser (name, age) values ('wangwu', 22);
```

## 主从复制

7、主节点创建同步账号
```postgresql
CREATE USER repl REPLICATION LOGIN ENCRYPTED PASSWORD '123456';
```

8、修改 pg_hba.conf
```text
# TYPE DATABASE USER ADDRESS METHOD
host all repl 0.0.0.0/0 md5
```

9、修改 postgresql.conf
> [按照 Cc 的说法](https://www.net-cc.com/posts/linux-software/postgresql/02-postgresql-master_slave-replication/)：主从服务器都要设置, 万一那天从服务器提升为主了呢
```text
listen_addresses = '*'
wal_level = replica
archive_mode = off
max_wal_senders = 32
wal_sender_timeout = 60s
wal_keep_segments = 64

max_replication_slots = 10 # (修改) 设置支持的复制槽数量
max_slot_wal_keep_size = 1GB # (修改) 设置复制槽保留的 wal 最大值,默认单位是 M

hot_standby = on
hot_standby_feedback = on # 如果有错误的数据复制向主进行反馈
# synchronous_commit = on  # 开启同步复制
```

10、重启主节点

11、从节点同步主节点数据
11.1、停止从节点

```shell
sudo service postgresql stop
```

11.2、在从节点上备份主节点数据

```shell
pg_basebackup -h <master-host> -U repl -p 5432 -F p  -X stream -v -P -R -D /data/pgsql -C -S slave01 -l slave01

# -D 指定备份数据存放的目录
# -F, --format=p|t 输出格式 (纯文本 (缺省值), tar压缩格式)
# -X, --wal-method=none|fetch|stream 按指定的模式包含必需的WAL日志文件
# -v, --verbose 输出详细的消息
# -P, --progress 显示进度信息
# -R, --write-recovery-conf 为复制写配置文件
# -C, --create-slot 创建复制槽
# -S, --slot=SLOTNAME 用于复制的槽名
# -l, --label=LABEL 设置备份标签

# 备份从节点原先的数据
mv /var/lib/postgresql/16/main /var/lib/postgresql/16/main.bak

# 使用主节点同步过来的数据代替从节点原先的数据
mv /data/pgsql /var/lib/postgresql/16/main

chown -R postgres:postgres -R /var/lib/postgresql/16/main
```

```postgresql
-- 复制槽状态检查
select * from pg_replication_slots;

-- 创建复制槽
select * from pg_create_physical_replication_slot('slave01');

-- 删除复制槽
select * from pg_drop_replication_slot('slave01');
```

11.2、启动从节点，并检查从节点状态
```shell
sudo service postgresql start
```

```postgresql
-- 节点状态检查
-- 查询结果为"f"表示主库, 't'表示从库
select pg_is_in_recovery();
```

```postgresql
-- 在主节点检查同步状态
select * from pg_stat_replication;
```

输出内容大概如下：
```shell
postgres=# select * from pg_stat_replication \g
-[ RECORD 1 ]----+------------------------------
pid              | 2299
usesysid         | 16388
usename          | repl
application_name | 16/main
client_addr      | 192.168.111.14
client_hostname  |
client_port      | 48412
backend_start    | 2023-10-13 03:11:24.319331+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/3000148
write_lsn        | 0/3000148
flush_lsn        | 0/3000148
replay_lsn       | 0/3000148
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2023-10-13 03:12:24.342696+00
```

12、尝试在主库插入数据，在从库查询。从库有记录表示主从复制配置完成。多配置几个从节点就是一主多从架构了。

---

**主节点宕机会发生什么？**

1、假设现在为一主二从的状态

```shell
postgres=# select * from pg_replication_slots;
-[ RECORD 1 ]-------+----------
slot_name           | slave01
plugin              |
slot_type           | physical
datoid              |
database            |
temporary           | f
active              | t
active_pid          | 2739
xmin                |
catalog_xmin        |
restart_lsn         | 0/7000330
confirmed_flush_lsn |
wal_status          | reserved
safe_wal_size       |
two_phase           | f
conflicting         |
-[ RECORD 2 ]-------+----------
slot_name           | slave02
plugin              |
slot_type           | physical
datoid              |
database            |
temporary           | f
active              | t
active_pid          | 3553
xmin                |
catalog_xmin        |
restart_lsn         | 0/7000330
confirmed_flush_lsn |
wal_status          | reserved
safe_wal_size       |
two_phase           | f
conflicting         |
```

2、让主节点下线

3、主节点下线后查看从节点状态
```shell
postgres=# select pg_is_in_recovery();
-[ RECORD 1 ]-----+--
pg_is_in_recovery | t
```

可以看到依然是从库，需要手动选择某一个从库提升为主库
```shell
cd /usr/lib/postgresql/16/bin
./pg_ctl promote -D /var/lib/postgresql/16/main
```

输出内容大概如下：
```shell
waiting for server to promote.... done
server promoted
```

再次检查该从节点状态
```shell
postgres=# select pg_is_in_recovery();
 pg_is_in_recovery
-------------------
 f
(1 row)
```

理论上来说可以这样子操作：

> 该从节点已经变成主节点了，接下来为对应的从节点创建复制槽
>
> ```postgresql
> select * from pg_create_physical_replication_slot('slave0X');
> 
> -- 查看复制槽状态
> select * from pg_replication_slots;
> ```
>
> 最后编辑其他从节点的 `postgresql.auto.conf` 文件，修改 `primary_conninfo` 为新的主节点地址，重启从节点即可。

但是我操作一番并未成功，可能需要再走一遍之前的步骤 11。

---

<br>

## 高可用部署

* PostgreSQL Streaming Replication 流复制（主从复制）
* PostgreSQL Streaming Replication + Replication Manager（repmgr）
* PostgreSQL Streaming Replication + PGPool
* Patroni

---


## 参考

**主从复制**
* https://www.net-cc.com/posts/linux-software/postgresql/02-postgresql-master_slave-replication/
* https://help.aliyun.com/zh/ecs/use-cases/build-a-primary-or-secondary-postgresql-architecture