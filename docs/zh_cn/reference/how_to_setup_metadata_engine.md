---
sidebar_label: 如何设置元数据引擎
sidebar_position: 3
slug: /databases_for_metadata
---
# JuiceFS 如何设置元数据引擎

:::tip 版本提示
本文档使用的环境变量 `META_PASSWORD` 是 JuiceFS v1.0 新增功能，旧版客户端需要升级后才能使用。
:::

通过阅读 [JuiceFS 的技术架构](../introduction/architecture.md) 和 [JuiceFS 如何存储文件](../reference/how_juicefs_store_files.md)，你会了解到 JuiceFS 被设计成了一种将数据和元数据独立存储的架构，通常来说，数据被存储在以对象存储为主的云存储中，而数据所对应的元数据则被存储在独立的数据库中，我们把这些支持存储元数据的数据库称为“元数据存储引擎”。

## 元数据与存储引擎

**元数据**至关重要，它记录着每一个文件的详细信息，名称、大小、权限、位置等等。特别是这种数据与元数据分离存储的文件系统，元数据的读写性能决定了文件系统实际的性能表现，而存储元数据的引擎是性能和可靠性最根本的决定因素。

JuiceFS 的元数据存储采用了多引擎设计。为了打造一个超高性能的云原生文件系统，JuiceFS 最先支持的是运行在内存上的键值数据库—— [Redis](https://redis.io)，这使得 JuiceFS 拥有十倍于 Amazon [EFS](https://aws.amazon.com/efs) 和 [S3FS](https://github.com/s3fs-fuse/s3fs-fuse) 的性能表现，[查看测试结果](../benchmark/benchmark.md)。

通过与社区用户积极互动，我们发现很多应用场景并不绝对依赖高性能，有时用户只是想临时找到一个方便的工具在云上可靠的迁移数据，或者只是想更简单的把对象存储挂载到本地小规模地使用。因此，JuiceFS 陆续开放了对 PostgreSQL、MySQL、MariaDB、TiKV 等更多数据库的支持（性能对比数据可参考[这里](../benchmark/metadata_engines_benchmark.md)）。

:::caution 特别提示
不论采用哪种数据库存储元数据，**务必确保元数据的安全**。元数据一旦损坏或丢失，将导致对应数据彻底损坏或丢失，甚至损毁整个文件系统。对于生产环境，应该始终选择具有高可用能力的数据库，与此同时，建议定期「[备份元数据](../administration/metadata_dump_load.md)」。
:::

## Redis

[Redis](https://redis.io/) 是基于内存的键值存储系统，在 BSD 协议下开源，可用于数据库、缓存和消息代理。

:::note 注意
JuiceFS 要求 Redis 4.0+ 版本
:::

### 创建文件系统

使用 Redis 作为元数据存储引擎时，通常使用以下格式访问数据库：

```shell
redis://[<username>:<password>@]<host>[:6379]/1
```

其中，`[]` 括起来的是可选项，其它部分为必选项。

- `username` 是 Redis 6.0 之后引入的，如果没有用户名可以忽略，但密码前面的 `:` 冒号需要保留，如 `redis://:password@host:6379/1`。
- `redis://` 协议头默认端口号为 `6379`，如果没有改变默认端口号可以不用填写，如 `redis://:password@host/1`。

例如，创建名为 `pics` 的文件系统，使用 Redis 的 `1` 号数据库存储元数据：

```shell
$ juicefs format --storage s3 \
    ...
    "redis://:mypassword@192.168.1.6:6379/1" \
    pics
```

安全起见，建议使用环境变量 `META_PASSWORD` 或 `REDIS_PASSWORD` 传递数据库密码，例如：

```shell
export META_PASSWORD=mypassword
```

然后就无需在元数据 URL 中设置密码了：

```shell
$ juicefs format --storage s3 \
    ...
    "redis://192.168.1.6:6379/1" \
    pics
```

:::note 说明
使用环境变量传递数据库密码也可以采用标准的 URL 格式，如：`"redis://:@192.168.1.6:6379/1"` 保留了用户名和密码之间的 `:` 以及 `@` 分隔符。
:::

### 挂载文件系统

```shell
sudo juicefs mount -d "redis://:mypassword@192.168.1.6:6379/1" /mnt/jfs
```

挂载文件系统也支持用 `META_PASSWORD` 或 `REDIS_PASSWORD` 环境变量传递密码：

```shell
$ export META_PASSWORD=mypassword
$ sudo juicefs mount -d "redis://192.168.1.6:6379/1" /mnt/jfs
```

:::tip 提示
如果需要在多台服务器上共享同一个文件系统，必须确保每台服务器都能访问到存储元数据的数据库。
:::

如果你自己维护 Redis 数据库，建议阅读 [Redis 最佳实践](../administration/metadata/redis_best_practices.md)。

## KeyDB

[KeyDB](https://keydb.dev/) 是 Redis 的开源分支，在开发上保持与 Redis 主线对齐。KeyDB 在 Redis 的基础上实现了多线程支持、更好的内存利用率和更大的吞吐量，另外还支持 [Active Replication](https://github.com/JohnSully/KeyDB/wiki/Active-Replication)，即 `Active Active` "双活"功能。

:::note 注意
KeyDB 的数据复制是异步的，使用 `Active Active` "双活"功能可能导致数据一致性问题，请务必充分验证、谨慎使用！
:::

在用于 JuiceFS 元数据存储时，KeyDB 与 Redis 的用法完全一致，这里不再赘述，请参考 [Redis](#redis) 部分使用。

## PostgreSQL

[PostgreSQL](https://www.postgresql.org/) 是功能强大的开源关系型数据库，有完善的生态和丰富的应用场景，也可以用来作为 JuiceFS 的元数据引擎。

许多云计算平台都提供托管的 PostgreSQL 数据库服务，也可以按照[使用向导](https://www.postgresqltutorial.com/postgresql-getting-started/)自己部署一个。

其他跟 PostgreSQL 协议兼容的数据库（比如 CockroachDB 等) 也可以这样使用。

### 创建文件系统

使用 PostgreSQL 作为元数据引擎时，需要使用如下的格式来指定参数：

```shell
postgres://<username>[:<password>]@<host>[:5432]/<database-name>[?parameters]
```

其中，`[]` 括起来的是可选项，其它部分为必选项。

例如：

```shell
$ juicefs format --storage s3 \
    ...
    "postgres://user:mypassword@192.168.1.6:5432/juicefs" \
    pics
```

更安全的做法是可以通过环境变量 `META_PASSWORD` 传递数据库密码：

```shell
$ export META_PASSWORD=mypassword
$ juicefs format --storage s3 \
    ...
    "postgres://user@192.168.1.6:5432/juicefs" \
    pics
```

### 挂载文件系统

```shell
sudo juicefs mount -d "postgres://user:mypassword@192.168.1.6:5432/juicefs" /mnt/jfs
```

挂载文件系统也支持用 `META_PASSWORD` 环境变量传递密码：

```shell
$ export META_PASSWORD=mypassword
$ sudo juicefs mount -d "postgres://user@192.168.1.6:5432/juicefs" /mnt/jfs
```

### 故障排除

JuiceFS 客户端默认采用 SSL 加密连接 PostgreSQL，如果连接时报错  `pq: SSL is not enabled on the server` 说明数据库没有启用 SSL。可以根据业务场景为 PostgreSQL 启用 SSL 加密，也可以在元数据 URL 中添加参数禁用加密验证：

```shell
$ juicefs format --storage s3 \
    ...
    "postgres://user@192.168.1.6:5432/juicefs?sslmode=disable" \
    pics
```

元数据 URL 中还可以附加更多参数，[查看详情](https://pkg.go.dev/github.com/lib/pq#hdr-Connection_String_Parameters)。

## MySQL

[MySQL](https://www.mysql.com/) 是受欢迎的开源关系型数据库之一，常被作为 Web 应用程序的首选数据库。

### 创建文件系统

使用 MySQL 作为元数据存储引擎时，通常使用以下格式访问数据库：

```shell
mysql://<username>[:<password>]@(<host>:3306)/<database-name>
```

:::note 注意
不要漏掉 URL 两边的 `()` 括号
:::

例如：

```shell
$ juicefs format --storage s3 \
    ...
    "mysql://user:mypassword@(192.168.1.6:3306)/juicefs" \
    pics
```

更安全的做法是可以通过环境变量 `META_PASSWORD` 传递数据库密码：

```shell
$ export META_PASSWORD=mypassword
$ juicefs format --storage s3 \
    ...
    "mysql://user@(192.168.1.6:3306)/juicefs" \
    pics
```

### 挂载文件系统

```shell
sudo juicefs mount -d "mysql://user:mypassword@(192.168.1.6:3306)/juicefs" /mnt/jfs
```

挂载文件系统也支持用 `META_PASSWORD` 环境变量传递密码：

```shell
$ export META_PASSWORD=mypassword
$ sudo juicefs mount -d "mysql://user@(192.168.1.6:3306)/juicefs" /mnt/jfs
```

更多 MySQL 数据库的地址格式示例，[点此查看](https://github.com/Go-SQL-Driver/MySQL/#examples)。

## MariaDB

[MariaDB](https://mariadb.org) 是 MySQL 的一个开源分支，由 MySQL 原始开发者维护并保持开源。

MariaDB 与 MySQL 高度兼容，在使用上也没有任何差别，创建和挂载文件系统时，保持与 MySQL 相同的语法。

例如：

```shell
$ juicefs format --storage s3 \
    ...
    "mysql://user:mypassword@(192.168.1.6:3306)/juicefs" \
    pics

$ sudo juicefs mount -d "mysql://user:mypassword@(192.168.1.6:3306)/juicefs" /mnt/jfs
```

通过环境变量传递密码的方式也完全一致：

```shell
$ export META_PASSWORD=mypassword
$ juicefs format --storage s3 \
    ...
    "mysql://user@(192.168.1.6:3306)/juicefs" \
    pics

$ sudo juicefs mount -d "mysql://user@(192.168.1.6:3306)/juicefs" /mnt/jfs
```

## SQLite

[SQLite](https://sqlite.org) 是全球广泛使用的小巧、快速、单文件、可靠、全功能的单文件 SQL 数据库引擎。

SQLite 数据库只有一个文件，创建和使用都非常灵活，用它作为 JuiceFS 元数据存储引擎时无需提前创建数据库文件，可以直接创建文件系统：

```shell
$ juicefs format --storage s3 \
    ...
    "sqlite3://my-jfs.db" \
    pics
```

以上命令会在当前目录创建名为 `my-jfs.db` 的数据库文件，请 **务必妥善保管** 这个数据库文件！

挂载文件系统：

```shell
sudo juicefs mount -d "sqlite3://my-jfs.db" /mnt/jfs/
```

请注意数据库文件的位置，如果不在当前目录，则需要指定数据库文件的绝对路径，比如：

```shell
sudo juicefs mount -d "sqlite3:///home/herald/my-jfs.db" /mnt/jfs/
```

:::note 注意
由于 SQLite 是一款单文件数据库，在不做特殊共享设置的情况下，只有数据库所在的主机可以访问它。对于多台服务器共享同一文件系统的情况，需要使用 Redis 或 MySQL 等数据库。
:::

## TiKV

[TiKV](https://github.com/tikv/tikv) 是一个分布式事务型的键值数据库，最初作为 [PingCAP](https://pingcap.com) 旗舰产品 [TiDB](https://github.com/pingcap/tidb) 的存储层而研发，现已独立开源并从 [CNCF](https://www.cncf.io/projects) 毕业。

TiKV 的测试环境搭建非常简单，使用官方提供的 TiUP 工具即可实现一键部署，具体可参见[这里](https://tikv.org/docs/5.1/concepts/tikv-in-5-minutes/)。生产环境一般需要至少三个节点来存储三份数据副本，部署步骤可以参考[官方文档](https://tikv.org/docs/5.1/deploy/install/install/)。

### 创建文件系统

使用 TiKV 作为元数据引擎时，需要使用如下格式来指定参数：

```shell
tikv://<pd_addr>[,<pd_addr>...]/<prefix>
```

其中 `prefix` 是一个用户自定义的字符串，当多个文件系统或者应用共用一个 TiKV 集群时，设置前缀可以避免混淆和冲突。示例如下：

```shell
$ juicefs format --storage s3 \
    ...
    "tikv://192.168.1.6:2379,192.168.1.7:2379,192.168.1.8:2379/jfs" \
    pics
```

### 挂载文件系统

```shell
sudo juicefs mount -d "tikv://192.168.1.6:6379,192.168.1.7:6379,192.168.1.8:6379/jfs" /mnt/jfs
```

## FoundationDB

即将推出......
