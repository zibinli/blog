![](https://github.com/zibinli/blog/blob/master/MySQL/image/2-0.png?raw=true)

正所谓理论造航母，现实小帆船。单有理论，不动手实践，学到的知识犹如空中楼阁。接下来，我们一起来看下如何一步步进行 MySQL Replication 的配置。

为 MySQL 服务器配置复制非常简单。但由于场景不同，基本的步骤还是有所差异。最基本的场景是新安装主库和备库，总得来说分为以下几步：
1. 在每台服务器上创建复制账号。
2. 配置主库和备库。
3. 通知备库连接到主库并从主库复制数据。

此外，由于主备部署需要多台服务器，但是这种要求对大多数人来说并不怎么友好，毕竟没有必要为了学习部署主备结构，多买个云服务器。因此，为了测试方便，我们通过 docker 容器技术在同台机器上部署多个容器，从而实现在一台机器上部署主备结构。

这里我们先假定大部分配置采用默认值，在主库和备库都是全新安装并且拥有同样的数据。接下来，我们将展示如何通过 docker 技术一步步进行复制配置。

此外，我们将推荐一些“安全配置”，以便在不清楚如何配置时，确保数据的安全。

### 1 部署 docker 环境
**1) 部署 docker**

什么？docker 还没部署？[赶紧参考这里配一个](http://www.runoob.com/docker/ubuntu-docker-install.html)，docker 都没玩，怎么和面试官吹水呀！

**2) 拉取 MySQL 镜像**
```
docker pull mysql:5.7
```
**3) 使用 mysql 镜像启动容器**
```
docker run -p 3339:3306 --name mysql-master -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7 # 启动 master 容器
docker run -p 3340:3306 --name mysql-slave -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7 # 启动 slave 容器
```
master 对外的端口是 3339，slave 对外的端口是 3340，我们在使用客户端连接要使用对应的端口连接对应 mysql。

**4) 使用命令查看正在运行的容器**
```
docker ps
```
![docker-ps](https://github.com/zibinli/blog/blob/master/MySQL/image/2-1.png?raw=true)

**5) 使用客户端连接工具测试丽连接 mysql**

![接测试](https://github.com/zibinli/blog/blob/master/MySQL/image/2-2.png?raw=true)

### 2 配置 Master 和 Slave
**1) 配置 master**
通过以下命令进入容器内部
```
docker exec -it mysql-master /bin/bash
```
a) 更新 apt-get 源
```
apt-get update
```

b) 安装 vim
```
apt-get install vim
```
c) 配置 my.cnf
```
vim /etc/mysql/my.cnf
// 在my.cnf 中添加如下配置
server-id=110 # 服务器 id，同一局域网内唯一
log-bin=/var/lib/mysql/mysql-bin # 二进制日志路径
```
d) 重启 mysql 服务使配置生效
```
service mysql restart
```
e) 启动容器
重启 mysql 服务时会使得 docker 容器停止，需要重启容器。
```
docker start mysql-master
```
f) 创建数据同步用户并授权
```
CREATE USER 'slave'@'%' IDENTIFIED BY '123456';
GRANT REPLICATION SLAVE,REPLICATION CLIENT ON *.* TO 'slave'@'%';
```

**2) 配置 slave**
通过以下命令进入容器内部
```
docker exec -it mysql-slave /bin/bash
```
a) 配置 my.cnf
```
vim /etc/mysql/my.cnf
// 在my.cnf 中添加如下配置
server-id=120 # 服务器 id，同一局域网内唯一
log-bin=/var/lib/mysql/mysql-bin # 二进制日志路径
relay_log=/path/to/logs/relay-bin # 中继日志路径
```

**3) 关联 master 和 slave**
配置完 master 和 slave，接下来就要让 master 和 slave 相关联。
回到我们的服务器，先找出 master 和 slave 容器的 IP，执行：
```
docker inspect --format='{{.NetworkSettings.IPAddress}}' mysql-master
```

![master IP](https://github.com/zibinli/blog/blob/master/MySQL/image/2-3.png?raw=true)

因此，我们知道了 mysql-master 容器的 IP 是：*172.17.0.3*。同样的方法，mysq-slave 容器的 IP 是：*172.17.0.4*。记住这两个值，后面的配置需要用到。

我们首先配置 master。在 master 容器内通过 *mysql -u root -p* 进入 MySQL 命令行，执行 *show master status;*

![master状态](https://github.com/zibinli/blog/blob/master/MySQL/image/2-4.png?raw=true)

上图中，File 和 Position 字段对应的值要记录下来，后续在 slave 配置时需要用到这两个值。要注意的是，记录完这两个值后，就不能在 master 库上做任何操作，否则会出现数据不同步的情况。

接下来配置 slave，同样的，在 slave 上进入 MySQL 命令行。然后执行下面语句：
```
change master to master_host='172.17.0.3', master_user='slave', master_password='123456', master_port=3306, master_log_file='mysql-bin.000001', master_log_pos=42852, master_connect_retry=30;
```

*change master to* 是 slave 配置 master 的命令，相关参数含义如下：
- master_host：master 的IP，就是我们上面获取的 IP 地址
- master_port：master 的端口号，也就是我们 master mysql 的端口号
- master_user：进行数据同步的用户
- master_password：同步用户的密码
- master_log_file：指定 slave 从 master 的哪个日志文件开始复制数据，也就是我们上面提到的 File 字段的值
- master_log_pos：从 master 日志文件的那个位置开始读，上面提到的 Position 字段的值
- master_connect_retry：重试时间间隔。单位是秒，默认 60

### 3 启动复制
配置完 slave 后，可以通过 *show slave status\G;*  查看 slave 的状态。

![lave 状态](https://github.com/zibinli/blog/blob/master/MySQL/image/2-5.png?raw=true)

正常情况下，刚配置完 slave 的 Slave_IO_Running 和 Slave_SQL_Runing 都是 NO，因为我们还没开启主从复制。使用 *start slave* 开启主从复制，然后再查下 slave 状态。

![开启主从复制后的 slave 状态](https://github.com/zibinli/blog/blob/master/MySQL/image/2-6.png?raw=true)

slave 的 Slave_IO_Running 和 Slave_SQL_Runing 都是 YES，说明主从复制已成功启动。此时，可以通过客户端能否成功复制数据。

我们在 master 新建 replication 库，然后观察 slave 库是否创建了 replication 库，如下图，表示复制成功。

![主从复制测试](https://github.com/zibinli/blog/blob/master/MySQL/image/2-7.png?raw=true)

另外，开启主从复制后，如果出现以下情况：
```
Slave_IO_Running: CONNECTING
Slave_SQL_RUNNING: Yes
```
表示开启主从复制后， slave 的 IO 进程连接 master 出现问题，一直在重试连接。我们可以根据 Last_IO_Error 的提示进行解决：
1. 网络不通。检查 IP、port。
2. 密码错误。检查配置的同步用户和密码是否正确。
3. pos 错误。检查 slave 配置的 Position 的值 与 master 是否一致。

### 4 从另一个服务器开始复制
前面的设置都是假定主备库均为刚刚安装好且都是默认的数据，也就是说两台服务器上数据相同，并且知道当前主库的二进制日志。但在实际环境中，大多数情况下是有一个一级运行了一段时间的主库，然后用一台新安装的备库与之同步，此时这台备库还没有数据。

有几种方法来初始化备库或者从其他服务器克隆数据到备库。包括从主库复制数据、从另外一台备库克隆数据，以及使用最近的一次备份来启动备库等。而这些方法都需要有三个条件来让主库与备库保持同步：
- 在某个时间点的主库的数据快照。
- 主库当前的**二进制日志文件**，和获得数据快照时在该二进制日志文件中的**偏移量**。我们把这两个值称为**日志文件坐标（log file coordinates）**。通过这两个值可以确定二进制日志的位置。可以通过 SHOW MASTER STATUS 命令来获取这些值。
- 从快照时间到现在的二进制日志。

下面是一些从别的服务器克隆备库的方法：
1. **使用冷备份**。最基本的方法是关闭主库，把数据复制到备库。重启主库后，会使用一个新的二进制日志文件，我们在备库通过执行 CHANGE MASTER TO 指向这个文件的起始处。不过这个方法的缺点很明显：在复制数据时需要关闭主库。
2. **使用热备份**。如果仅使用了 MyISAM 表，可以在主库运行时使用 mysqlhotcopy 或 rsync 来复制数据。
3. **使用 mysqldump**。如果只包含 InnoDB 表，可以使用以下命令来转储主库数据并将其加载到备库，然后设置相应的二进制日志坐标：*mysqldump --single-transaction --all-databases --master-data=1 --host=server1 | mysql --host=server2*。选项 --single-transaction 使得转储的数据为事务开始前的数据。如果使用的是非事务型表，可以使用 --lock-all-tables 选项来获得所有表的一致性转储。
4. **使用快照或备份**。只要知道对应的二进制日志坐标，就可以使用主库的快照或者备份来初始化备库。（如果使用备份，需要确保从备份的时间点开始的主库二进制日志都要存在）。只需要把备份或快照恢复到备库，然后使用 CHANGE MASTER TO 指定二进制日志的坐标。
5. **使用 Percona Xtrabackup**。Percona 的 Xtrabackup 是一款开源的热备份工具。它能够在备份时不阻塞服务器的操作，因此可以在不影响主库的情况下设置备库。可以通过克隆主库或另一个已存在的备库的方式来建立备库。
6. **使用另外的备库**。可以使用任何一种克隆或拷贝技术从任意一台备库上将数据克隆到另外一台服务器。但是如果使用的是 mysqldump，--master-data 选项就会不起作用。此外，不能使用 SHOW MASTER STATUS 来获得主库的二进制日志坐标，而是在获取快照时使用 SHOW SLAVE STATUS 来获取备库在主库上的执行位置。使用另外的备库进行数据克隆最大的缺点是，如果这台备库的数据已经和主库不同步，克隆得到的就是脏数据。

### 5 推荐的复制配置
我们知道，MySQL 的复制有许多参数可以控制，其中一些会对数据安全和性能产生影响。这里，我们介绍一种“安全配置”，可以最小化问题发生的概率。

在主库上二进制日志最重要的选项是 sync_binlog：
> sync_binlog=1
如果开启该选项，MySQL 每次在提交事务前会将二进制日志同步到磁盘上，保证在服务器崩溃时不会丢失时间。如果禁止该选项，服务器会少做一些工作，但二进制日志文件可能在服务器崩溃时损坏或丢失信息。在一个不需要作为主库的备库上 ，该选项会带来不必要的开销。要注意的是，它只适用于二进制日志，而非中继日志。

如果无法接受服务器崩溃导致表损坏，推荐使用 InnoDB。MyISAM 表在备库服务器崩溃重启后，可能已经处于不一致状态。

如果使用 InnoDB，推荐设置如下选项：
```
innodb_flush_logs_at_trx_commit=1 # 每次事务提交时，将 log buffer 写入到日志文件并刷新到磁盘。默认值为 1
innodb_safe_binlog
```

**明确指定二进制日志文件的名称**。当服务器间转移文件、克隆新的备库、转储备份或者其他场景下，如果以服务器名来命名二进制日志可能会导致很多问题。因此，我们需要给 log_bin 选项指定一个参数。
```
log_bin=/var/lib/mysql/mysql-bin
```
在备库上，同样开启如下培训，为中继日志指定绝对路径：
```
relay_log=/path/to/logs/relay-bin
skip_slave_start
read_only
```
通过设置 relay_log 可以避免中继日志文件基于机器名来命名，防止之前提到的可能在主库上发生的问题。而 skip_slave_start 选项能够阻止备库在崩溃后自动启动复制，以留出时间修复可能发生的问题。read_only 选项可以阻止大部分用户更改非临时表。

### 6 小结
1. 复制初始化配置三部曲：创建账号、配置主备库、备库连接到主库开始复制；
2. 从已有服务器复制时，可用热备份或 mysqldump 命令进行备份；
3. 在不确定相关配置时，选择最安全的配置准没错；