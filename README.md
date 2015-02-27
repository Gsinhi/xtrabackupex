# 使用Percona XtraBackup热备和恢复MySQL

# 环境

  `fedora 19`  `mysql 5.5`  `xtrabackuper 版本  2.1 `

### 官方说明及文档
1. 安装包适用说明

    ```
    RPM packages for RHEL 5 and RHEL 6 (including compatible distributions such as CentOS and Oracle Enterprise Linux)
    Debian packages for Debian and Ubuntu
    Generic .tar.gz binary packages
    ```

2. 文档

    2.1.  [基础文档](http://www.percona.com/doc/percona-xtrabackup/2.1/)
    2.2.  [源码安装文档](http://www.percona.com/doc/percona-xtrabackup/2.1/installation/compiling_xtrabackup.html)
    2.3.  [官方脚本文档](http://www.percona.com/doc/percona-xtrabackup/2.1/innobackupex/innobackupex_script.html)

### 先决条件(In RPM-based distributions )

```sh
yum install cmake gcc gcc-c++ libaio libaio-devel automake autoconf bzr bison libtool ncurses-devel zlib-devel
```

```
sudo yum install libgcrypt libgcrypt-devel
```

```
sudo yum install perl perl-devel perl-Time-HiRes perl-DBD-MySQL
```

----

### 安装
1. 仅说明 `source` 安装方式,

2. 安装包
    1.1. 自行下载 [源码包地址](https://code.launchpad.net/percona-xtrabackup)
    1.2. 使用bzr获取

    ```sh
    bzr branch lp:percona-xtrabackup/2.1
    ```

4. 该脚本需要其它针对性的代码库，下列提供值或别名, 选择适用Value, ( `示例环境` )

    | Value | Alias | Source tarball download link | Description |
    | -------- | -------- | -------- | -------- |
    | innodb51 | plugin | [Download](http://s3.amazonaws.com/percona.com/downloads/community/mysql-5.1.59.tar.gz) | 在MySQL5.1建立InnoDB的插件 |
    | innodb55 | 5.5 | [Download](http://s3.amazonaws.com/percona.com/downloads/community/mysql-5.5.17.tar.gz) | 在MySQL5.5 建立 InnoDB 支持 |
    | xtradb51 | xtradb | [Download](http://s3.amazonaws.com/percona.com/downloads/community/mysql-5.1.59.tar.gz) | 在 Percona Server XtraDB5.1 建立支持 |
    | xtradb55 | xtradb55 | [Download](http://s3.amazonaws.com/percona.com/downloads/community/mysql-5.5.16.tar.gz) | 在 Percona Server XtraDB5.5 建立支持 |
    | innodb56 | 5.6 | [Download](http://s3.amazonaws.com/percona.com/downloads/community/mysql-5.6.10.tar.gz) | 在MySQL5.6 建立 InnoDB 支持 |

5. 编译安装

    ```sh
    sudo AUTO_DOWNLOAD="yes" ./utils/build.sh innodb55
    ```
    会自动下载: mysql安装包,
    如果当前目录下存在MYSQL包,可以不加 AUTO_DOWNLOAD="yes"


6. 完成

    ```sh
    sudo ln -s /usr/local/xtrabackup/innobackupex /usr/bin/innobackupex
    ```

    ```sh
    sudo ln -s /usr/local/xtrabackup/src/xtrabackup_innodb55 /usr/bin/xtrabackup_55
    ```

    按照官方文档, 上述操作也可换作 `copy` 操作

    ```sh
    sudo cp /usr/local/xtrabackup/innobackupex /usr/bin/innobackupex
    ```

    ```sh
    sudo cp /usr/local/xtrabackup/src/xtrabackup_innodb55 /usr/bin/xtrabackup_55
    ```

### 备份脚本
1. 前提

    ```sh
    innobackupex --user=DBUSER --password=SECRET /path/to/backup/dir/
    ```

    可能需要创建具有所需的完整备份的最小权限的数据库用户
    该数据库用户需要对要备份的表/数据库具有以下权限:
    - RELOAD和LOCK TABLES（除非-无锁指定的选项），以FLUSH TABLES WITH READ LOCK前，开始复制文件和
    - REPLICATION CLIENT为了获得二进制日志的位置，
    - CREATE TABLESPACE，以导入表（见恢复单个表）和
    - SUPER以启动/停止在复制环境中的从属线程。


2. 完整备份

    ```sh
    innobackupex --defaults-file=MYSQL_CNF  --no-timestamp --user=USERNAME --password=PASSWD --slave-info
                 --databases="$DB" /opt/backupdata/full
    ```

3. 准备一个完整备份

    ```sh
    innobackupex --apply-log /opt/backupdata/full --use-memory=1G
    ```

    `use-memory` 默认100M

4. 还原一个完整备份

    ```sh
    innobackupex --copy-back /opt/backupdata/full
    ```

    注:
        1). 使用前需关闭mysql服务器
        2). 完成后需通过 `chown` 设置mysql数据目录的执行权限


5. 增量备份
    5.1. 增量备份需要做一个完整的备份为基础, 并且仅针对存储引擎 `innodb` 有效, 对存储引擎 `MyISAM` 仍是完整备份

    ```sh
    innobackupex --defaults-file=MYSQL_CNF  --no-timestamp --user=USERNAME --password=PASSWD --slave-info
              --databases="$DB" --incremental --incremental-basedir=/opt/backupdata/full /opt/backupdata/rec
    ```

6. 还原增量备份
    6.1 准备完整备份基础 (开启 BEGIN)

    ```sh
    innobackupex --apply-log --redo-only /opt/backupdata/full
    ```

    6.2 准备增量备份 (准备 EXEC)

    ```sh
    innobackupex --apply-log --redo-only /opt/backupdata/full --incremental-dir=/opt/backupdata/rec
    ```

    6.3 还原准备 (回滚 ROOLBACK)

    ```sh
    innobackupex --apply-log /opt/backupdata/full
    ```

    6.4 还原已准备的备份到原始目录 (提交 COMMIT)

    ```sh
    innobackupex --copy-back /opt/backupdata/full
    ```

7. 选项参考

    | Options | Description |
    | -------- | -------- |
    | apply-log | 开启备份目录准备事务 |
    | copy-back | 将已准备的备份目录复制到原始目录 |
    | databases | 备份的数据库名集合, eg: wiki, kutongji |
    | defaults-file | MySQL 配置文件 /etc/my.cnf |
    | host | 连接到数据库时指定的主机地址 |
    | incremental | 通知xtrabackup创建增量备份 |
    | incremental-basedir | 指定包含完全备份是基本数据集的增量备份的目录 |
    | incremental-dir | 指定了增量备份将与完全备份，使一个新的完整备份相结合的目录 |
    | no-lock | 使用此选项可以使用FLUSH TABLES WITH READ LOCK禁用表锁 |
    | no-timestamp | 没有时间戳 |
    | password | 连接到数据库时指定的密码 |
    | port | 连接到数据库时指定的端口 |
    | redo-only | 准备合并所有增量除了完全备份 |
    | slave-info | 打印的主服务器的二进制日志的位置和名称 |
    | socket | SOCKET连接 /tmp/mysql.socket |
    | stream | 备份的数据流格式,格式: tar,xbstream |
    | use-memory | 准备备份指定xtrabackup的内存 |
    | user | MySQL连接到服务器时使用的用户名 |
    | tmpdir | 临时文件将被存储的位置 |
    | throttle | 选项接受一个整数参数，指定每秒I/O操作数 |
    | tables-file | 该选项接受一个字符串参数，指定其中有形式database.table，每行一个名称的列表文件 |
