---
title: "安装 MySQL 5.7(CentOS 7)"
date: 2024-06-14
categories: ['安装']
draft: false
description: 本文描述了在CentOS 7.9操作系统上安装MySQL 5.7.44，同样适用于CentOS 7系列的其他版本，也适用于Red Hat Enterprise Linux 7以及Oracle 7。本文介绍了两种安装方法，一种是使用官方提供的rpm包进行安装，另一种是使用源代码编译安装。
---

安装目标：

- [x] 安装一个 MySQL 服务端实例，版本是 5.7.44。
- [x] 两种安装方法，一种是使用官方提供的 rpm 包进行安装，另一种是使用源代码编译安装。

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

---

## 编译安装

1. 安装依赖
```bash
yum install cmake make gcc-c++ openssl openssl-devel ncurses ncurses-devel perl -y
```

2. 安装依赖 boost

浏览器打开[https://sourceforge.net/projects/boost/files/boost/1.59.0/](https://sourceforge.net/projects/boost/files/boost/1.59.0/)，下载 boost 1.59.0 版本。
![3.png](/images/install_mysql57/3.png)
将下载的 boost_1_59_0.tar.gz 放到 CentOS 7 服务器的/usr/local 目录下，解压，删除压缩包。
```bash
tar -zxf /usr/local/boost_1_59_0.tar.gz -C /usr/local
rm -f /usr/local/boost_1_59_0.tar.gz
```

3. 下载 MySQL 源代码

打开 [MySQL官方下载地址](https://downloads.mysql.com/archives/community/)，Product Version 选择 5.7.44，Operating System 选择 Source Code，OS Version 选择 All Operating Systems (Generic) (Architecture Independent)。点击 Compressed TAR Archive 一栏的 Download 按钮，开始下载。
![4.png](/images/install_mysql57/4.png)

4. 解压

MySQL 源代码压缩包放到/usr/local/src 目录下，解压到同级目录，删除压缩包。
```bash
tar -zxf /usr/local/src/mysql-5.7.44.tar.gz -C /usr/local/src
rm -f /usr/local/src/mysql-5.7.44.tar.gz
cd /usr/local/src/mysql-5.7.44/
```

5. 编译并安装到/usr/local/mysql 目录下
```bash
mkdir bld
cd bld
cmake .. -DCMAKE_INSTALL_PREFIX=/usr/local/mysql -DWITH_INNODB_MEMCACHED=1 -DWITH_BOOST=/usr/local/boost_1_59_0 
make
make install
```

6. 创建用户 mysql 和用户组 mysql
```bash
groupadd mysql
useradd -r -g mysql -s /bin/false mysql
```

7. 修改目录的属主和属组
```bash
chown -R mysql:mysql /usr/local/mysql
```

8. 将/usr/local/mysql/bin 下的可执行文件（包含服务端和客户端及工具）加入环境变量
```bash
echo 'export MYSQL_HOME=/usr/local/mysql' > /etc/profile.d/mysql.sh
echo 'export PATH=$PATH:$MYSQL_HOME/bin' >> /etc/profile.d/mysql.sh
source /etc/profile.d/mysql.sh
```

9. 验证上一步操作是否成功
```console
[root@localhost bld]# mysql -V
mysql  Ver 14.14 Distrib 5.7.44, for Linux (x86_64) using  EditLine wrapper
```

10. 创建数据目录和存放 pid 的目录，并且修改属主属组
```bash
mkdir /var/run/mysqld /var/lib/mysql
chown mysql:mysql /var/run/mysqld /var/lib/mysql
```

11. 创建错误日志输出文件，并且修改属主属组
```bash
touch /var/log/mysqld.log
chown mysql:mysql /var/log/mysqld.log
```

12. 创建配置文件/etc/my.cnf
```ini
# For advice on how to change settings please see
# http://dev.mysql.com/doc/refman/5.7/en/server-configuration-defaults.html

[mysqld]
#
# Remove leading # and set to the amount of RAM for the most important data
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
innodb_buffer_pool_size =256M
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
basedir=/usr/local/mysql
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

13. 初始化数据目录
```bash
mysqld --initialize-insecure --user=mysql
```

14. 复制$MYSQL_HOME/support-files/mysql.server 到/etc/init.d/mysqld，这是官方提供的启动、停止脚本，我们把它放在/etc/init.d 目录下，就可以使用 service 命令启动、停止 MySQL 服务器。
```bash
cp $MYSQL_HOME/support-files/mysql.server /etc/init.d/mysqld
```

15. 赋予/etc/init.d/mysqld 可执行权限
```bash
chmod +x /etc/init.d/mysqld
```

16. 设置开机自启动 MySQL 服务器
```bash
chkconfig --add mysqld
```

17. 启动 MySQL 服务器
```console
[root@localhost bld]# service mysqld start
Starting MySQL. SUCCESS!
```

18. 直接登录，当前'root'@'localhost'用户无密码，因为在第 15 步的时候使用--initialize-insecure 参数。
```bash
mysql -uroot -p
```

19. 因为超级用户'root'@'localhost'还没有密码，所以登录后第一件事就是设置超级用户'root'@'localhost'的正式密码
```bash
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4!';
```

20. 创建用户'example'@'%'且设置密码，密码永不过期
```bash
mysql> CREATE USER 'example'@'%' IDENTIFIED BY 'MyNewPass4!' PASSWORD EXPIRE NEVER;
```

21. 赋予用户'example'@'%'全部权限
```bash
mysql> GRANT ALL ON *.* TO 'example'@'%';
```

22. 退出客户端
```bash
mysql> exit
```

