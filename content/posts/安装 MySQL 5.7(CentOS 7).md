---
title: "安装 MySQL 5.7(CentOS 7)"
date: 2024-06-14
categories: ['安装']
draft: false
description: 本文描述了在CentOS 7.9操作系统上安装MySQL 5.7.44，同样适用于CentOS 7系列的其他版本，也适用于Red Hat Enterprise Linux 7以及Oracle 7。
---

安装目标：

- [x] 安装一个 MySQL 服务端实例，版本是 5.7.44。

---

前置条件：

- [x] 待安装 MySQL 的机器能访问 yum 仓库。

---

## 通过官方 rpm 包安装
> 重要：5.7.44 版本是 5.7 系列的最后一个版本，不会再更新了，详见[官方文档](https://dev.mysql.com/doc/relnotes/mysql/5.7/en/news-5-7-44.html)。

1. 打开 MySQL 官方下载地址[https://downloads.mysql.com/archives/community/](https://downloads.mysql.com/archives/community/)

Product Version 选择 5.7.44，Operating System 选择 Red Hat Enterprise Linux / Oracle Linux，OS Version 选择 Red Hat Enterprise Linux 7 / Oracle Linux 7 (x86, 64-bit)。点击 RPM Bundle 一栏的 Download 按钮，开始下载。
![图片1.png](/images/install_mysql57/1.png)

2. 将上一步下载的文件上传到装有 CentOS 7 的服务器上，放哪里都可以。在本文中，我将文件放在了/root/mysql_bundle 目录下。此处的目录仅仅是用来放 rpm 安装包，最终的安装目录不是它。
```console
[root@localhost mysql_bundle]# ll
total 557332
-rw-r--r--. 1 root root 570705920 Jun 13 18:30 mysql-5.7.44-1.el7.x86_64.rpm-bundle.tar
```

3. 解压
```console
[root@localhost mysql_bundle]# tar -xf mysql-5.7.44-1.el7.x86_64.rpm-bundle.tar
```

4. 安装
```console
[root@localhost mysql_bundle]# yum install mysql-community-{server,client,common,libs}-*
```
下面是 yum 命令执行后的依赖分析，可以看到除了从官网下载的以外，还需要依赖很多安装包，比如 net-tools、perl 等等。输入 y 后回车，同意安装。
```text
Dependencies Resolved

===========================================================================================================
 Package                 Arch   Version             Repository                                        Size
===========================================================================================================
Installing:
 mysql-community-client  x86_64 5.7.44-1.el7        /mysql-community-client-5.7.44-1.el7.x86_64      120 M
 mysql-community-common  x86_64 5.7.44-1.el7        /mysql-community-common-5.7.44-1.el7.x86_64      2.8 M
 mysql-community-libs    x86_64 5.7.44-1.el7        /mysql-community-libs-5.7.44-1.el7.x86_64         11 M
     replacing  mariadb-libs.x86_64 1:5.5.68-1.el7
 mysql-community-libs-compat
                         x86_64 5.7.44-1.el7        /mysql-community-libs-compat-5.7.44-1.el7.x86_64 6.0 M
     replacing  mariadb-libs.x86_64 1:5.5.68-1.el7
 mysql-community-server  x86_64 5.7.44-1.el7        /mysql-community-server-5.7.44-1.el7.x86_64      796 M
Installing for dependencies:
 net-tools               x86_64 2.0-0.25.20131004git.el7
                                                    os                                               306 k
 perl                    x86_64 4:5.16.3-299.el7_9  updates                                          8.0 M
 perl-Carp               noarch 1.26-244.el7        os                                                19 k
 perl-Encode             x86_64 2.51-7.el7          os                                               1.5 M
 perl-Exporter           noarch 5.68-3.el7          os                                                28 k
 perl-File-Path          noarch 2.09-2.el7          os                                                26 k
 perl-File-Temp          noarch 0.23.01-3.el7       os                                                56 k
 perl-Filter             x86_64 1.49-3.el7          os                                                76 k
 perl-Getopt-Long        noarch 2.40-3.el7          os                                                56 k
 perl-HTTP-Tiny          noarch 0.033-3.el7         os                                                38 k
 perl-PathTools          x86_64 3.40-5.el7          os                                                82 k
 perl-Pod-Escapes        noarch 1:1.04-299.el7_9    updates                                           52 k
 perl-Pod-Perldoc        noarch 3.20-4.el7          os                                                87 k
 perl-Pod-Simple         noarch 1:3.28-4.el7        os                                               216 k
 perl-Pod-Usage          noarch 1.63-3.el7          os                                                27 k
 perl-Scalar-List-Utils  x86_64 1.27-248.el7        os                                                36 k
 perl-Socket             x86_64 2.010-5.el7         os                                                49 k
 perl-Storable           x86_64 2.45-3.el7          os                                                77 k
 perl-Text-ParseWords    noarch 3.29-4.el7          os                                                14 k
 perl-Time-HiRes         x86_64 4:1.9725-3.el7      os                                                45 k
 perl-Time-Local         noarch 1.2300-2.el7        os                                                24 k
 perl-constant           noarch 1.27-2.el7          os                                                19 k
 perl-libs               x86_64 4:5.16.3-299.el7_9  updates                                          690 k
 perl-macros             x86_64 4:5.16.3-299.el7_9  updates                                           44 k
 perl-parent             noarch 1:0.225-244.el7     os                                                12 k
 perl-podlators          noarch 2.5.1-3.el7         os                                               112 k
 perl-threads            x86_64 1.87-4.el7          os                                                49 k
 perl-threads-shared     x86_64 1.43-6.el7          os                                                39 k

Transaction Summary
===========================================================================================================
Install  5 Packages (+28 Dependent packages)

Total size: 947 M
Total download size: 12 M
Is this ok [y/d/N]: 
```

5. 修改配置文件/etc/my.cnf

修改前
```ini
# For advice on how to change settings please see
# http://dev.mysql.com/doc/refman/5.7/en/server-configuration-defaults.html

[mysqld]
#
# Remove leading # and set to the amount of RAM for the most important data
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
# innodb_buffer_pool_size = 128M
#
# Remove leading # to turn on a very important data integrity option: logging
# changes to the binary log between backups.
# log_bin
#
# Remove leading # to set options mainly useful for reporting servers.
# The server defaults are faster for transactions and fast SELECTs.
# Adjust sizes as needed, experiment to find the optimal values.
# join_buffer_size = 128M
# sort_buffer_size = 2M
# read_rnd_buffer_size = 2M
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock

# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0

log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
```
修改后
```ini
# For advice on how to change settings please see
# http://dev.mysql.com/doc/refman/5.7/en/server-configuration-defaults.html

[mysqld]
#
# Remove leading # and set to the amount of RAM for the most important data
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
innodb_buffer_pool_size =1G
innodb_buffer_pool_instances=1
innodb_log_file_size=256M
innodb_lru_scan_depth=128
innodb_io_capacity=10000
innodb_io_capacity_max=16000
innodb_status_output_locks=on
innodb_print_all_deadlocks=on
innodb_undo_tablespaces=10

character_set_server=utf8
#
# Remove leading # to turn on a very important data integrity option: logging
# changes to the binary log between backups.
log-bin=master-bin
server_id=1
#
# Remove leading # to set options mainly useful for reporting servers.
# The server defaults are faster for transactions and fast SELECTs.
# Adjust sizes as needed, experiment to find the optimal values.
join_buffer_size = 128M
sort_buffer_size = 8M
# read_rnd_buffer_size = 2M
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock

# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0

log-error=/var/log/mysqld.log
log_timestamps=SYSTEM
pid-file=/var/run/mysqld/mysqld.pid

log_output=TABLE
slow_query_log=ON
long_query_time=2
tls_version=TLSv1.2

sql_mode='STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION'

[client]
socket=/var/lib/mysql/mysql.sock
```

6. 启动服务
```bash
systemctl start mysqld
```

7. 设置防火墙
```bash
firewall-cmd --add-service=mysql
firewall-cmd --add-service=mysql --permanent
```

8. 获取超级用户'root'@'localhost'的临时密码
```bash
grep 'temporary password' /var/log/mysqld.log
```

9. 使用临时密码登录
```bash
mysql -uroot -p
```

10. 设置超级用户'root'@'localhost'的正式密码
```text
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4!';
```

11. 创建用户'example'@'%'且设置密码，密码永不过期
```text
mysql> CREATE USER 'example'@'%' IDENTIFIED BY 'MyNewPass4!' PASSWORD EXPIRE NEVER;
```

12. 赋予用户'example'@'%'全部权限
```text
mysql> GRANT ALL ON *.* TO 'example'@'%';
```

13. 退出客户端
```text
mysql> exit
```
