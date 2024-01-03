---
title: 安装 Hive 集群(Rocky Linux 9.2)
date: 2024-01-03
categories: ['安装']
draft: false
---

安装目标：

- [x] Hive 集群启用 Kerberos 认证。
- [x] Hive 由 2 个 HiveServer2 和 2 个 MetaStore 组成集群。
- [x] Hive 版本是 3.1.3。

---

前置条件：

- [x] 已按照[安装 Hadoop 集群(Rocky Linux 9.2)](../安装hadoop集群rocky-linux-9.2/)安装 Hadoop。
- [x] 已安装 MySQL。

---

## 准备用户
> 注意⚠️：下面的命令是在主机 ipa-server 上执行的。

1. 身份认证
```console
[root@ipa-server ~]# kinit admin
Password for admin@XUWANGWEI.TEST: 
```

2. 新增用户 hive
```console
ipa user-add hive --first=hive --last=hadoop --shell=/bin/bash --gidnumber=1532600004
su - hive
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 0600 ~/.ssh/authorized_keys
exit
ipa user-mod hive --sshpubkey="$( cat ~hive/.ssh/id_rsa.pub )"
```
## 共享 Hive 配置目录
> 注意⚠️：下面的命令是在主机 ipa-server 上执行的。

1. 修改 NFS 配置文件/etc/exports，使得主机 ipa-server 上的/etc/hive 目录共享给其他主机。
```bash
echo '/etc/hive 192.168.2.0/24(rw,sync,no_root_squash)' >> /etc/exports
```

2. 创建 Hive 配置目录并设置属主属组
```bash
mkdir /etc/hive
chown hive:hadoop /etc/hive
chmod 755 /etc/hive
```

3. 刷新 NFS 的配置
```bash
exportfs -rv
```

4. 在 IPA Server 上创建挂载配置
```bash
ipa automountkey-add default auto.direct --key=/etc/hive --info="-fstype=nfs4,rw ipa-server:/etc/hive"
```
## 安装 Hive
> 注意⚠️：下面的命令需要在主机 machine1~machine5 上执行。

1. 下载、解压、创建别名
```bash
cd /opt
curl -O https://dlcdn.apache.org/hive/hive-3.1.3/apache-hive-3.1.3-bin.tar.gz
tar -zxf apache-hive-3.1.3-bin.tar.gz
ln -s apache-hive-3.1.3-bin hive
```

2. 添加环境变量
```bash
echo 'export HIVE_HOME=/opt/hive' > /etc/profile.d/hive.sh
echo 'export PATH=$PATH:$HIVE_HOME/bin' >> /etc/profile.d/hive.sh
echo 'export HIVE_CONF_DIR=/etc/hive' >> /etc/profile.d/hive.sh
source /etc/profile.d/hive.sh
```

3. 下载 MySQL JDBC 驱动到 $HIVE_HOME/lib 目录下
```bash
curl -O --output-dir $HIVE_HOME/lib https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.0.33/mysql-connector-j-8.0.33.jar
```

4. 创建数据目录并设置属主属组
```bash
chown -R -H hive:hadoop $HIVE_HOME
chmod 755 $HIVE_HOME
mkdir /var/lib/hive
chown hive:hadoop /var/lib/hive
chmod 750 /var/lib/hive
```

5. 创建配置目录并设置属主属组
```bash
mkdir /etc/hive
chown hive:hadoop /etc/hive
chmod 755 /etc/hive
```

6. 创建日志目录并设置属主属组
```bash
mkdir /var/log/hive
chown hive:hadoop /var/log/hive
chmod 755 /var/log/hive
```

7. 共享 ipa-server 的/etc/hive 目录
```bash
systemctl reload autofs
```
## 准备 Kerberos principals 和 keytab files
> 注意⚠️：下面的命令是在主机 ipa-server 上执行的。

1. 创建principal
```bash
ipa service-add hive/machine1.xuwangwei.test@XUWANGWEI.TEST --requires-pre-auth=true
ipa service-add hive/machine2.xuwangwei.test@XUWANGWEI.TEST --requires-pre-auth=true
ipa service-add hive/machine3.xuwangwei.test@XUWANGWEI.TEST --requires-pre-auth=true
ipa service-add hive/machine4.xuwangwei.test@XUWANGWEI.TEST --requires-pre-auth=true
ipa service-add hive/machine5.xuwangwei.test@XUWANGWEI.TEST --requires-pre-auth=true
```

2. 导出principal的密钥
```bash
ipa-getkeytab --principal=HTTP/machine1.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hive/hive.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=HTTP/machine2.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hive/hive.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=HTTP/machine3.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hive/hive.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=HTTP/machine4.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hive/hive.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=HTTP/machine5.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hive/hive.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=hive/machine1.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hive/hive.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=hive/machine2.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hive/hive.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=hive/machine3.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hive/hive.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=hive/machine4.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hive/hive.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=hive/machine5.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hive/hive.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=hive@XUWANGWEI.TEST --keytab=/etc/hive/hive.keytab --enctypes=aes256-sha1,aes128-sha1
```

3. 修改keytab的属主属组和权限
```bash
chown hive:hadoop /etc/hive/*.keytab
chmod 640 /etc/hive/*.keytab
```
## 配置 Hive

1. 在MySQL数据库中创建用户hive，创建数据库metastore_db。这些信息会放在配置文件hive-site.xml中。
```console
# 在安装MySQL的主机上输入
mysql -uroot -p
# 成功登录后，先创建数据库metastore_db，再创建用户hive，最后把metastore_db的所有权限都授权给用户hive
mysql> CREATE DATABASE metastore_db CHARACTER SET utf8mb4;
mysql> CREATE USER 'hive'@'%' IDENTIFIED BY 'ifpx8#2c4Kv65';
mysql> GRANT ALL ON metastore_db.* TO 'hive'@'%';
mysql> quit;
```

2. 将数据库 metastore_db 的密码存储到/etc/hive/hive.jceks
```console
[root@machine1 ~]# su - hive -c "hadoop credential create javax.jdo.option.ConnectionPassword -provider jceks://file/etc/hive/hive.jceks"
WARNING: You have accepted the use of the default provider password
by not configuring a password in one of the two following locations:
    * In the environment variable HADOOP_CREDSTORE_PASSWORD
    * In a file referred to by the configuration entry
      hadoop.security.credstore.java-keystore-provider.password-file.
Please review the documentation regarding provider passwords in
the keystore passwords section of the Credential Provider API
Continuing with the default provider password.

Enter alias password: 
Enter alias password again: 
javax.jdo.option.ConnectionPassword has been successfully created.
Provider jceks://file/etc/hive/hive.jceks was updated.
```

3. 创建 hive-site.xml
```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!--
   Licensed to the Apache Software Foundation (ASF) under one or more
   contributor license agreements.  See the NOTICE file distributed with
   this work for additional information regarding copyright ownership.
   The ASF licenses this file to You under the Apache License, Version 2.0
   (the "License"); you may not use this file except in compliance with
   the License.  You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
-->
<configuration>
    <property>
        <name>system:user.name</name>
        <value>${user.name}</value>
    </property>
    <property>
        <name>system:java.io.tmpdir</name>
        <value>/var/lib/hive</value>
    </property>

    <property>
        <name>hive.metastore.uris</name>
        <value>thrift://machine4.xuwangwei.test:9083,thrift://machine5.xuwangwei.test:9083</value>
    </property>
    <property>
        <name>hive.metastore.kerberos.keytab.file</name>
        <value>/etc/hive/hive.keytab</value>
    </property>
    <property>
        <name>hive.metastore.kerberos.principal</name>
        <value>hive/_HOST@XUWANGWEI.TEST</value>
    </property>
    <property>
        <name>hive.metastore.client.kerberos.principal</name>
        <value>hive/_HOST@XUWANGWEI.TEST</value>
    </property>
    <property>
        <name>hive.metastore.sasl.enabled</name>
        <value>true</value>
    </property>
    <property>
        <name>hive.metastore.db.type</name>
        <value>mysql</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.cj.jdbc.Driver</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mysql://mysql.xuwangwei.test:3306/metastore_db?characterEncoding=UTF-8&amp;characterSetResults=UTF-8&amp;connectionTimeZone=Asia/Shanghai</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>hive</value>
    </property>
    <property>
        <name>hadoop.security.credential.provider.path</name>
        <value>jceks://file/etc/hive/hive.jceks</value>
    </property>
    <property>
        <name>hive.metastore.port</name>
        <value>9083</value>
    </property>
    <property>
        <name>hive.metastore.schema.verification</name>
        <value>false</value>
    </property>
    <property>
        <name>hive.metastore.schema.verification.record.version</name>
        <value>false</value>
    </property>
    <property>
        <name>hive.metastore.warehouse.dir</name>
        <value>/user/hive/warehouse</value>
    </property>

    <property>
        <name>hive.server2.webui.port</name>
        <value>10002</value>
        <description>The port the HiveServer2 WebUI will listen on. This can beset to 0 or a negative integer to disable the web UI</description>
    </property>
    <property>
        <name>hive.server2.webui.use.spnego</name>
        <value>false</value>
    </property>
    <property>
        <name>hive.server2.webui.spnego.keytab</name>
        <value>/etc/hive/hive.keytab</value>
    </property>
    <property>
        <name>hive.server2.webui.spnego.principal</name>
        <value>HTTP/_HOST@XUWANGWEI.TEST</value>
    </property>

    <property>
        <name>hive.server2.authentication</name>
        <value>kerberos</value>
    </property>
    <property>
        <name>hive.server2.thrift.sasl.qop</name>
        <value>auth-conf</value>
    </property>
    <property>
        <name>hive.server2.thrift.port</name>
        <value>10000</value>
        <description>Port number of HiveServer2 Thrift interface when hive.server2.transport.mode is 'binary'.</description>
    </property>
    <property>
        <name>hive.server2.authentication.kerberos.keytab</name>
        <value>/etc/hive/hive.keytab</value>
    </property>
    <property>
        <name>hive.server2.authentication.kerberos.principal</name>
        <value>hive/_HOST@XUWANGWEI.TEST</value>
    </property>
    <property>
        <name>hive.server2.authentication.client.kerberos.principal</name>
        <value>hive/_HOST@XUWANGWEI.TEST</value>
    </property>
    <property>
        <name>hive.server2.authentication.spnego.keytab</name>
        <value>/etc/hive/hive.keytab</value>
    </property>
    <property>
        <name>hive.server2.authentication.spnego.principal</name>
        <value>HTTP/_HOST@XUWANGWEI.TEST</value>
    </property>
    <property>
        <name>hive.compute.query.using.stats</name>
        <value>false</value>
    </property>
    <property>
        <name>hive.exec.parallel</name>
        <value>true</value>
        <description>Whether to execute jobs in parallel</description>
    </property>
    <property>
        <name>hive.execution.engine</name>
        <value>mr</value>
        <description>
            Expects one of [mr, tez, spark].
            Chooses execution engine. Options are: mr (Map reduce, default), tez, spark. While MR
            remains the default engine for historical reasons, it is itself a historical engine
            and is deprecated in Hive 2 line. It may be removed without further warning.
        </description>
    </property>

    <property>
        <name>hive.server2.support.dynamic.service.discovery</name>
        <value>true</value>
    </property>
    <property>
        <name>hive.server2.zookeeper.namespace</name>
        <value>hiveserver2</value>
    </property>
    <property>
        <name>hive.zookeeper.quorum</name>
        <value>machine1.xuwangwei.test:2181,machine2.xuwangwei.test:2181,machine3.xuwangwei.test:2181,machine4.xuwangwei.test:2181,machine5.xuwangwei.test:2181</value>
    </property>
</configuration>
```

4. 修改配置文件 $HADOOP_CONF_DIR/core-site.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->

<!-- Put site-specific property overrides in this file. -->

<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://xuwangwei</value>
  </property>
  <property>
    <name>hadoop.tmp.dir</name>
    <value>/var/lib/hadoop</value>
  </property>
  <property>
    <name>hadoop.zk.address</name>
    <value>machine1.xuwangwei.test:2181,machine2.xuwangwei.test:2181,machine3.xuwangwei.test:2181,machine4.xuwangwei.test:2181,machine5.xuwangwei.test:2181</value>
  </property>
  <property>
    <name>ha.zookeeper.quorum</name>
    <value>machine1.xuwangwei.test:2181,machine2.xuwangwei.test:2181,machine3.xuwangwei.test:2181,machine4.xuwangwei.test:2181,machine5.xuwangwei.test:2181</value>
  </property>
  <property>
    <name>hadoop.security.authentication</name>
    <value>kerberos</value>
  </property>
  <property>
    <name>hadoop.proxyuser.hive.hosts</name>
    <value>*</value>
  </property>
  <property>
    <name>hadoop.proxyuser.hive.groups</name>
    <value>*</value>
  </property>
</configuration>
```

5. 使用 hdfs@XUWANGWEI@TEST 帐号进行 Kerberos 认证
```bash
su - hdfs -c "kinit -kt /etc/hadoop/hdfs.keytab hdfs"
```

6. 在 HDFS 上创建目录
```bash
su - hdfs -c 'hadoop fs -mkdir -p /user/hive/warehouse'
su - hdfs -c 'hadoop fs -chown -R hive:hadoop /user/hive'
su - hdfs -c 'hadoop fs -chmod 755 /user/hive'
su - hdfs -c 'hadoop fs -chmod 775 /user/hive/warehouse'
```

7. 初始化 MetaStore 使用的数据库
```bash
su - hive -c 'schematool -dbType mysql -initSchema'
```

8. 使 core-silte 中刚修改的配置生效
```bash
su - hdfs -c "hdfs dfsadmin -refreshSuperUserGroupsConfiguration"
echo 'export KRB5CCNAME=FILE:/tmp/krb5cc_$UID' >> ~yarn/.bashrc
su - yarn -c "kinit -kt /etc/hadoop/yarn.keytab yarn"
su - yarn -c "yarn rmadmin -refreshSuperUserGroupsConfiguration"
```
## 启动 Hive

1. 启动 MetaStore
```bash
su - hive -c "ssh machine4 'nohup hive --service metastore 1>/dev/null 2>&1 &'"
su - hive -c "ssh machine5 'nohup hive --service metastore 1>/dev/null 2>&1 &'"
```

2. 启动 HiveServer2
```bash
su - hive -c "ssh machine4 'nohup hive --service hiveserver2 1>/dev/null 2>&1 &'"
su - hive -c "ssh machine5 'nohup hive --service hiveserver2 1>/dev/null 2>&1 &'"
```
## 使用 Beeline 连接 Hive 服务

1. 添加 hive 用户的环境变量
```bash
echo 'export KRB5CCNAME=FILE:/tmp/krb5cc_$UID' >> ~hive/.bashrc
```

2. 使用 hive@XUWANGWEI@TEST 帐号进行 Kerberos 认证
```bash
su - hive -c "kinit -kt /etc/hive/hive.keytab hive"
```

3. 使用 Beeline 连接 Hive 集群
```bash
su - hive -c "beeline"
beeline> !connect jdbc:hive2://machine1.xuwangwei.test:2181,machine2.xuwangwei.test:2181,machine3.xuwangwei.test:2181,machine4.xuwangwei.test:2181,machine5.xuwangwei.test:2181/;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2;saslQop=auth-conf;auth=KERBEROS;principal=hive/_HOST@XUWANGWEI.TEST;
```

4. 连接成功，日志如下所示
```text
Connecting to jdbc:hive2://machine1.xuwangwei.test:2181,machine2.xuwangwei.test:2181,machine3.xuwangwei.test:2181,machine4.xuwangwei.test:2181,machine5.xuwangwei.test:2181/;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2;saslQop=auth-conf;auth=KERBEROS;principal=hive/_HOST@XUWANGWEI.TEST;
24/01/03 15:12:56 [main]: INFO jdbc.HiveConnection: Connected to machine4.xuwangwei.test:10000
Connected to: Apache Hive (version 3.1.3)
Driver: Hive JDBC (version 3.1.3)
Transaction isolation: TRANSACTION_REPEATABLE_READ
```
