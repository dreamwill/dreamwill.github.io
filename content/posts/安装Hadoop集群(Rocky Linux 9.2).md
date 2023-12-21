---
title: 安装 Hadoop 集群(Rocky Linux 9.2)
date: 2023-12-10
categories: ['安装']
draft: false
---

安装目标：

- [x] Hadoop 集群启用 Kerberos 认证。
- [x] HDFS 由 3 个 NameNode、5 个 DataNode 组成高可用（High Availability）集群。
- [x] MapReduce 由 1 个 JobHistory Server 组成。
- [x] YARN 由 2 个 ResourceManager、5 个 NodeManager 组成高可用集群。
- [x] Hadoop 版本是 3.3.6，ZooKeeper 版本是 3.7.1。

---

前置条件：

- [x] 已安装JDK 8。
- [x] 已安装 FreeIPA。可以参考[安装 FreeIPA(Rocky Linux 9.2)](../安装freeiparockylinux-9.2/)。
- [x] 已编译 container-executor。可以参考[安装 Hadoop 过程中遇到的问题与解决方案](../安装hadoop过程中遇到的问题与解决方案/)。
- [x] 已安装 openssl-1.1.1。可以参考[安装 Hadoop 过程中遇到的问题与解决方案](../安装hadoop过程中遇到的问题与解决方案/)。

---

|  | ipa-server | ipa-client | namenode | zkfc | journalnode | datanode | jobhistory server | resource manager | node manager | zookeeper |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| ipa-server | ✔️ |  |  |  |  |  |  |  |  |  |
| machine1 |  | ✔️ | ✔️ | ✔️ | ✔️ | ✔️ | ✔️ |  | ✔️ | ✔️ |
| machine2 |  | ✔️ | ✔️ | ✔️ | ✔️ | ✔️ |  |  | ✔️ | ✔️ |
| machine3 |  | ✔️ | ✔️ | ✔️ | ✔️ | ✔️ |  |  | ✔️ | ✔️ |
| machine4 |  | ✔️ |  |  |  | ✔️ |  | ✔️ | ✔️ | ✔️ |
| machine5 |  | ✔️ |  |  |  | ✔️ |  | ✔️ | ✔️ | ✔️ |


---

目录布局

|  | User:Group | Permissions | 说明 |
| --- | --- | --- | --- |
| /opt/hadoop | root:hadoop | 755 | 环境变量 HADOOP_HOME 的值，是 Hadoop 的家目录 |
| /etc/hadoop | root:hadoop | 755 | 环境变量 HADOOP_CONF_DIR 的值，是 Hadoop 的配置目录 |
| /var/lib/hadoop | root:hadoop | 770 | core-site.xml 中 hadoop.tmp.dir 的值，是 Hadoop 的数据目录 |
| /var/log/hadoop | hdfs:hadoop | 775 | 环境变量 HADOOP_LOG_DIR 的值，是 Hadoop 的日志目录 |
| /opt/zookeeper | zookeeper:zookeeper | 755 | 环境变量 ZOO_KEEPER_HOME 的值，是 ZooKeeper 的家目录 |
| /etc/zookeeper | zookeeper:zookeeper | 755 | 环境变量 ZOOCFGDIR 的值，是 ZooKeeper 的配置目录 |
| /var/lib/zookeeper | zookeeper:zookeeper | 750 | zoo.cfg 中 dataDir 的值，是 ZooKeeper 的数据目录 |
| /var/log/zookeeper | zookeeper:zookeeper | 755 | 环境变量 ZOO_LOG_DIR 的值，是 ZooKeeper 的日志目录 |

## 准备用户
> 注意⚠️：下面的命令是在主机ipa-server上执行的。

1. 身份认证
```console
[root@ipa-server ~]# kinit admin
Password for admin@XUWANGWEI.TEST: 
```

2. 新增用户组hadoop
```console
[root@ipa-server ~]# ipa group-add hadoop --gid=1532600004
```

3. 新增用户hdfs
```console
# 添加用户hdfs
[root@ipa-server ~]# ipa user-add hdfs --first=hdfs --last=hadoop --shell=/bin/bash --gidnumber=1532600004
# 生成用户hdfs的ssh公钥和私钥
[root@ipa-server ~]# su - hdfs
[hdfs@ipa-server ~]$ ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
[hdfs@ipa-server ~]$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
[hdfs@ipa-server ~]$ chmod 0600 ~/.ssh/authorized_keys
[hdfs@ipa-server ~]$ exit
# 上传用户hdfs的ssh公钥
[root@ipa-server ~]# ipa user-mod hdfs --sshpubkey="$( cat ~hdfs/.ssh/id_rsa.pub )"
```

4. 新增用户yarn
```console
ipa user-add yarn --first=yarn --last=hadoop --shell=/bin/bash --gidnumber=1532600004
su - yarn
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 0600 ~/.ssh/authorized_keys
exit
ipa user-mod yarn --sshpubkey="$( cat ~yarn/.ssh/id_rsa.pub )"
```

5. 新增用户mapred
```console
ipa user-add mapred --first=mapred --last=hadoop --shell=/bin/bash --gidnumber=1532600004
su - mapred
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 0600 ~/.ssh/authorized_keys
exit
ipa user-mod mapred --sshpubkey="$( cat ~mapred/.ssh/id_rsa.pub )"
```

6. 新增用户zookeeper
```console
ipa user-add zookeeper --first=zookeeper --last=hadoop --shell=/bin/bash
su - zookeeper
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 0600 ~/.ssh/authorized_keys
exit
ipa user-mod zookeeper --sshpubkey="$( cat ~zookeeper/.ssh/id_rsa.pub )"
```

---

## 共享家目录、Hadoop 和 ZooKeeper 配置目录
> 注意⚠️：下面的命令是在主机ipa-server上执行的。

1. 设置防火墙
```bash
firewall-cmd --add-service={nfs,rpc-bind,mountd}
firewall-cmd --add-service={nfs,rpc-bind,mountd} --permanent
```

2. 修改NFS配置文件/etc/exports，使得主机ipa-server上的/home目录、/etc/hadoop目录和/etc/zookeeper目录共享给其他主机。
```text
/home    192.168.2.0/24(rw,sync,no_root_squash)
/etc/hadoop    192.168.2.0/24(rw,sync,no_root_squash)
/etc/zookeeper 192.168.2.0/24(rw,sync,no_root_squash)
```

3. 创建 Hadoop 和 ZooKeeper 配置目录并设置属主属组
```bash
mkdir /etc/hadoop
chown root:hadoop /etc/hadoop
chmod 755 /etc/hadoop
mkdir /etc/zookeeper
chown zookeeper:zookeeper /etc/zookeeper
chmod 755 /etc/zookeeper
```

4. 启动NFS服务
```bash
systemctl restart rpcbind
systemctl restart nfs-server
systemctl enable rpcbind
systemctl enable nfs-server
```

5. 在IPA Server上创建挂载配置
```bash
ipa automountkey-add default auto.direct --key=/home --info="-fstype=nfs4,rw ipa-server:/home"
ipa automountkey-add default auto.direct --key=/etc/hadoop --info="-fstype=nfs4,rw ipa-server:/etc/hadoop"
ipa automountkey-add default auto.direct --key=/etc/zookeeper --info="-fstype=nfs4,rw ipa-server:/etc/zookeeper"
```

---

## 安装Hadoop和ZooKeeper
> 注意⚠️：下面的命令需要在主机machine1~machine5上执行。

1. 关闭防火墙
```bash
systemctl stop firewalld
systemctl disable firewalld
```

2. 下载和解压
```bash
cd /opt
curl -O https://dlcdn.apache.org/zookeeper/zookeeper-3.7.1/apache-zookeeper-3.7.1-bin.tar.gz
tar -zxf apache-zookeeper-3.7.1-bin.tar.gz
curl -O https://dlcdn.apache.org/hadoop/common/hadoop-3.3.6/hadoop-3.3.6.tar.gz
tar -zxf hadoop-3.3.6.tar.gz
```

3. 创建别名
```bash
ln -s apache-zookeeper-3.7.1-bin zookeeper
ln -s hadoop-3.3.6 hadoop
```

4. 添加环境变量
```bash
echo 'export ZOO_KEEPER_HOME=/opt/zookeeper' >> /etc/profile.d/zookeeper.sh
echo 'export PATH=$PATH:$ZOO_KEEPER_HOME/bin' >> /etc/profile.d/zookeeper.sh
echo 'export ZOOCFGDIR=/etc/zookeeper' >> /etc/profile.d/zookeeper.sh
echo 'export ZOO_LOG_DIR=/var/log/zookeeper' >> /etc/profile.d/zookeeper.sh
source /etc/profile.d/zookeeper.sh
echo 'export HADOOP_HOME=/opt/hadoop' >> /etc/profile.d/hadoop.sh
echo 'export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin' >> /etc/profile.d/hadoop.sh
echo 'export HADOOP_CONF_DIR=/etc/hadoop' >> /etc/profile.d/hadoop.sh
echo 'export HADOOP_LOG_DIR=/var/log/hadoop' >> /etc/profile.d/hadoop.sh
echo 'export HADOOP_OPTS="-Djava.security.auth.login.config=/etc/hadoop/jaas.conf"' >> /etc/profile.d/hadoop.sh
source /etc/profile.d/hadoop.sh
```

5. 创建数据目录并设置属主属组
```bash
chown -R -H zookeeper:zookeeper $ZOO_KEEPER_HOME
mkdir /var/lib/zookeeper
chown zookeeper:zookeeper /var/lib/zookeeper
chmod 750 /var/lib/zookeeper
mkdir /var/log/zookeeper
chown zookeeper:zookeeper /var/log/zookeeper
chmod 755 /var/log/zookeeper
chown -R -H root:hadoop $HADOOP_HOME
chmod 755 $HADOOP_HOME
chown root:hadoop $HADOOP_HOME/bin/container-executor
chmod 6050 $HADOOP_HOME/bin/container-executor 
mkdir /var/lib/hadoop
chown root:hadoop /var/lib/hadoop
chmod 770 /var/lib/hadoop
mkdir /var/log/hadoop
chown hdfs:hadoop /var/log/hadoop
chmod 775 /var/log/hadoop
```

6. 创建 Hadoop 和 ZooKeeper 配置目录并设置属主属组
```bash
mkdir /etc/hadoop
chown root:hadoop /etc/hadoop
chmod 755 /etc/hadoop
mkdir /etc/zookeeper
chown zookeeper:zookeeper /etc/zookeeper
chmod 755 /etc/zookeeper
```

7. 客户端挂载家目录、/etc/hadoop 和/etc/zookeeper 目录
```bash
ipa-client-automount --location=default
```

8. 复制配置文件至共享目录
> 注意⚠️：下面的命令是在主机 machine1 上执行的，目的是把默认配置文件复制到共享目录中。

```bash
cp -a -r /opt/hadoop/etc/hadoop/* /etc/hadoop
chown root:hadoop /etc/hadoop/container-executor.cfg
chmod 0400 /etc/hadoop/container-executor.cfg
cp -a -r /opt/zookeeper/conf/* /etc/zookeeper
```

---

## 准备Kerberos principals和keytab files
> 注意⚠️：下面的命令是在主机ipa-server上执行的。

1. 创建principal
```bash
ipa service-add hdfs/machine1.xuwangwei.test@XUWANGWEI.TEST --requires-pre-auth=true
ipa service-add HTTP/machine1.xuwangwei.test@XUWANGWEI.TEST --requires-pre-auth=true
ipa service-add yarn/machine1.xuwangwei.test@XUWANGWEI.TEST --requires-pre-auth=true
ipa service-add mapred/machine1.xuwangwei.test@XUWANGWEI.TEST --requires-pre-auth=true
ipa service-add hdfs/machine2.xuwangwei.test@XUWANGWEI.TEST --requires-pre-auth=true
ipa service-add HTTP/machine2.xuwangwei.test@XUWANGWEI.TEST --requires-pre-auth=true
ipa service-add yarn/machine2.xuwangwei.test@XUWANGWEI.TEST --requires-pre-auth=true
ipa service-add hdfs/machine3.xuwangwei.test@XUWANGWEI.TEST --requires-pre-auth=true
ipa service-add HTTP/machine3.xuwangwei.test@XUWANGWEI.TEST --requires-pre-auth=true
ipa service-add yarn/machine3.xuwangwei.test@XUWANGWEI.TEST --requires-pre-auth=true
ipa service-add hdfs/machine4.xuwangwei.test@XUWANGWEI.TEST --requires-pre-auth=true
ipa service-add HTTP/machine4.xuwangwei.test@XUWANGWEI.TEST --requires-pre-auth=true
ipa service-add yarn/machine4.xuwangwei.test@XUWANGWEI.TEST --requires-pre-auth=true
ipa service-add hdfs/machine5.xuwangwei.test@XUWANGWEI.TEST --requires-pre-auth=true
ipa service-add HTTP/machine5.xuwangwei.test@XUWANGWEI.TEST --requires-pre-auth=true
ipa service-add yarn/machine5.xuwangwei.test@XUWANGWEI.TEST --requires-pre-auth=true
ipa service-add zookeeper/machine1.xuwangwei.test@XUWANGWEI.TEST --requires-pre-auth=true
ipa service-add zookeeper/machine2.xuwangwei.test@XUWANGWEI.TEST --requires-pre-auth=true
ipa service-add zookeeper/machine3.xuwangwei.test@XUWANGWEI.TEST --requires-pre-auth=true
ipa service-add zookeeper/machine4.xuwangwei.test@XUWANGWEI.TEST --requires-pre-auth=true
ipa service-add zookeeper/machine5.xuwangwei.test@XUWANGWEI.TEST --requires-pre-auth=true
```

2. 导出principal的密钥
```bash
ipa-getkeytab --principal=hdfs/machine1.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hadoop/hdfs.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=HTTP/machine1.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hadoop/hdfs.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=hdfs/machine2.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hadoop/hdfs.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=HTTP/machine2.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hadoop/hdfs.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=hdfs/machine3.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hadoop/hdfs.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=HTTP/machine3.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hadoop/hdfs.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=hdfs/machine4.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hadoop/hdfs.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=HTTP/machine4.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hadoop/hdfs.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=hdfs/machine5.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hadoop/hdfs.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=HTTP/machine5.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hadoop/hdfs.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=HTTP/machine1.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hadoop/yarn.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=yarn/machine1.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hadoop/yarn.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=HTTP/machine2.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hadoop/yarn.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=yarn/machine2.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hadoop/yarn.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=HTTP/machine3.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hadoop/yarn.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=yarn/machine3.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hadoop/yarn.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=HTTP/machine4.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hadoop/yarn.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=yarn/machine4.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hadoop/yarn.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=HTTP/machine5.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hadoop/yarn.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=yarn/machine5.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hadoop/yarn.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=mapred/machine1.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/hadoop/mapred.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=hdfs@XUWANGWEI.TEST --keytab=/etc/hadoop/hdfs.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=yarn@XUWANGWEI.TEST --keytab=/etc/hadoop/yarn.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=mapred@XUWANGWEI.TEST --keytab=/etc/hadoop/mapred.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=zookeeper/machine1.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/zookeeper/zookeeper.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=zookeeper/machine2.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/zookeeper/zookeeper.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=zookeeper/machine3.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/zookeeper/zookeeper.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=zookeeper/machine4.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/zookeeper/zookeeper.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=zookeeper/machine5.xuwangwei.test@XUWANGWEI.TEST --keytab=/etc/zookeeper/zookeeper.keytab --enctypes=aes256-sha1,aes128-sha1
ipa-getkeytab --principal=zookeeper@XUWANGWEI.TEST --keytab=/etc/zookeeper/zookeeper.keytab --enctypes=aes256-sha1,aes128-sha1
```

3. 修改keytab的属主属组和权限
```bash
chown root:hadoop /etc/hadoop/*.keytab
chmod 640 /etc/hadoop/*.keytab
chown root:zookeeper /etc/zookeeper/*.keytab
chmod 640 /etc/zookeeper/*.keytab
```

---

## 生成自签名 SSL/TLS 证书
> 注意⚠️：下面的命令是在主机ipa-server上执行的。

1. 切换至 root 用户家目录
```bash
cd /root
```

2. 生成自签名证书
```bash
openssl req -newkey rsa:2048 -keyout domain1.key -x509 -days 365 -out domain1.crt -subj="/C=CN/ST=Jiangsu/L=Changzhou/CN=machine1.xuwangwei.test" -noenc
openssl req -newkey rsa:2048 -keyout domain2.key -x509 -days 365 -out domain2.crt -subj="/C=CN/ST=Jiangsu/L=Changzhou/CN=machine2.xuwangwei.test" -noenc
openssl req -newkey rsa:2048 -keyout domain3.key -x509 -days 365 -out domain3.crt -subj="/C=CN/ST=Jiangsu/L=Changzhou/CN=machine3.xuwangwei.test" -noenc
openssl req -newkey rsa:2048 -keyout domain4.key -x509 -days 365 -out domain4.crt -subj="/C=CN/ST=Jiangsu/L=Changzhou/CN=machine4.xuwangwei.test" -noenc
openssl req -newkey rsa:2048 -keyout domain5.key -x509 -days 365 -out domain5.crt -subj="/C=CN/ST=Jiangsu/L=Changzhou/CN=machine5.xuwangwei.test" -noenc
```

2. 转化成 PKCS12 格式
```bash
openssl pkcs12 -inkey domain1.key -in domain1.crt -export -out domain1.p12 -password pass:xuwangwei3306
openssl pkcs12 -inkey domain2.key -in domain2.crt -export -out domain2.p12 -password pass:xuwangwei3306
openssl pkcs12 -inkey domain3.key -in domain3.crt -export -out domain3.p12 -password pass:xuwangwei3306
openssl pkcs12 -inkey domain4.key -in domain4.crt -export -out domain4.p12 -password pass:xuwangwei3306
openssl pkcs12 -inkey domain5.key -in domain5.crt -export -out domain5.p12 -password pass:xuwangwei3306
```

3. 通过keytool把 5 个证书和对应私钥全部导入hadoop.p12
```bash
keytool -importkeystore -srckeystore domain1.p12 -srcstoretype pkcs12 -srcstorepass xuwangwei3306 -srcalias 1 -destkeystore hadoop.p12 -deststoretype pkcs12 -deststorepass xuwangwei3306 -destalias m1
keytool -importkeystore -srckeystore domain2.p12 -srcstoretype pkcs12 -srcstorepass xuwangwei3306 -srcalias 1 -destkeystore hadoop.p12 -deststoretype pkcs12 -deststorepass xuwangwei3306 -destalias m2
keytool -importkeystore -srckeystore domain3.p12 -srcstoretype pkcs12 -srcstorepass xuwangwei3306 -srcalias 1 -destkeystore hadoop.p12 -deststoretype pkcs12 -deststorepass xuwangwei3306 -destalias m3
keytool -importkeystore -srckeystore domain4.p12 -srcstoretype pkcs12 -srcstorepass xuwangwei3306 -srcalias 1 -destkeystore hadoop.p12 -deststoretype pkcs12 -deststorepass xuwangwei3306 -destalias m4
keytool -importkeystore -srckeystore domain5.p12 -srcstoretype pkcs12 -srcstorepass xuwangwei3306 -srcalias 1 -destkeystore hadoop.p12 -deststoretype pkcs12 -deststorepass xuwangwei3306 -destalias m5
```

4. 复制到配置目录
```bash
cp hadoop.p12 /etc/hadoop/
chown root:hadoop /etc/hadoop/hadoop.p12
```

---

## 配置Hadoop
> 注意⚠️：下面的命令只需在主机machine1上执行。

1. 修改配置文件$HADOOP_CONF_DIR/core-site.xml
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
</configuration>
```

2. 修改配置文件$HADOOP_CONF_DIR/hdfs-site.xml
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
    <name>dfs.nameservices</name>
    <value>xuwangwei</value>
  </property>
  <property>
    <name>dfs.ha.namenodes.xuwangwei</name>
    <value>nn1,nn2,nn3</value>
  </property>
  <property>
    <name>dfs.namenode.rpc-address.xuwangwei.nn1</name>
    <value>machine1.xuwangwei.test:8020</value>
  </property>
  <property>
    <name>dfs.namenode.rpc-address.xuwangwei.nn2</name>
    <value>machine2.xuwangwei.test:8020</value>
  </property>
  <property>
    <name>dfs.namenode.rpc-address.xuwangwei.nn3</name>
    <value>machine3.xuwangwei.test:8020</value>
  </property>
  <property>
    <name>dfs.namenode.http-address.xuwangwei.nn1</name>
    <value>machine1.xuwangwei.test:9870</value>
  </property>
  <property>
    <name>dfs.namenode.http-address.xuwangwei.nn2</name>
    <value>machine2.xuwangwei.test:9870</value>
  </property>
  <property>
    <name>dfs.namenode.http-address.xuwangwei.nn3</name>
    <value>machine3.xuwangwei.test:9870</value>
  </property>
  <property>
    <name>dfs.client.failover.proxy.provider.xuwangwei</name>
    <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
  </property>
  <property>
    <name>dfs.ha.fencing.methods</name>
    <value>sshfence</value>
  </property>
  <property>
    <name>dfs.ha.fencing.ssh.private-key-files</name>
    <value>/home/hdfs/.ssh/id_rsa</value>
  </property>
  <property>
    <name>dfs.namenode.shared.edits.dir</name>
    <value>qjournal://machine1.xuwangwei.test:8485;machine2.xuwangwei.test:8485;machine3.xuwangwei.test:8485/xuwangwei</value>
  </property>
  <property>
    <name>dfs.journalnode.edits.dir</name>
    <value>${hadoop.tmp.dir}/dfs/journalnode/</value>
  </property>
  <property>
    <name>dfs.ha.nn.not-become-active-in-safemode</name>
    <value>true</value>
  </property>
  <property>
    <name>dfs.ha.automatic-failover.enabled</name>
    <value>true</value>
  </property>
  <!-- start for kerberos -->
  <property>
    <name>dfs.block.access.token.enable</name>
    <value>true</value>
  </property>
  <property>
    <name>dfs.namenode.keytab.file</name>
    <value>/etc/hadoop/hdfs.keytab</value>
  </property>
  <property>
    <name>dfs.namenode.kerberos.principal</name>
    <value>hdfs/_HOST@XUWANGWEI.TEST</value>
  </property>
  <property>
    <name>dfs.namenode.kerberos.internal.spnego.principal</name>
    <value>HTTP/_HOST@XUWANGWEI.TEST</value>
  </property>
  <property>
    <name>dfs.journalnode.keytab.file</name>
    <value>/etc/hadoop/hdfs.keytab</value>
  </property>
  <property>
    <name>dfs.journalnode.kerberos.principal </name>
    <value>hdfs/_HOST@XUWANGWEI.TEST</value>
  </property>
  <property>
    <name>dfs.journalnode.kerberos.internal.spnego.principal</name>
    <value>HTTP/_HOST@XUWANGWEI.TEST</value>
  </property>
  <property>
    <name>dfs.data.transfer.protection</name>
    <value>authentication</value>
  </property>
  <property>
    <name>dfs.http.policy</name>
    <value>HTTPS_ONLY</value>
  </property>
  <property>
    <name>dfs.datanode.keytab.file</name>
    <value>/etc/hadoop/hdfs.keytab</value>
  </property>
  <property>
    <name>dfs.datanode.kerberos.principal</name>
    <value>hdfs/_HOST@XUWANGWEI.TEST</value>
  </property>
  <property>
    <name>dfs.web.authentication.kerberos.keytab</name>
    <value>/etc/hadoop/hdfs.keytab</value>
  </property>
  <property>
    <name>dfs.web.authentication.kerberos.principal</name>
    <value>HTTP/_HOST@XUWANGWEI.TEST</value>
  </property>
  <!-- end for kerberos -->
</configuration>
```

3. 修改配置文件$HADOOP_CONF_DIR/yarn-site.xml
```xml
<?xml version="1.0"?>
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
<configuration>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <property>
    <name>yarn.nodemanager.env-whitelist</name>
    <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_HOME,PATH,LANG,TZ,HADOOP_MAPRED_HOME</value>
  </property>
  <property>
    <name>yarn.resourcemanager.ha.enabled</name>
    <value>true</value>
  </property>
  <property>
    <name>yarn.resourcemanager.cluster-id</name>
    <value>xuwangwei</value>
  </property>
  <property>
    <name>yarn.resourcemanager.ha.rm-ids</name>
    <value>rm1,rm2</value>
  </property>
  <property>
    <name>yarn.resourcemanager.hostname.rm1</name>
    <value>machine4.xuwangwei.test</value>
  </property>
  <property>
    <name>yarn.resourcemanager.hostname.rm2</name>
    <value>machine5.xuwangwei.test</value>
  </property>
  <!-- start for kerberos -->
  <property>
    <name>yarn.resourcemanager.principal</name>
    <value>yarn/_HOST@XUWANGWEI.TEST</value>
  </property>
  <property>
    <name>yarn.resourcemanager.webapp.spnego-principal</name>
    <value>HTTP/_HOST@XUWANGWEI.TEST</value>
  </property>
  <property>
    <name>yarn.resourcemanager.keytab</name>
    <value>/etc/hadoop/yarn.keytab</value>
  </property>
  <property>
    <name>yarn.resourcemanager.webapp.spnego-keytab-file</name>
    <value>/etc/hadoop/yarn.keytab</value>
  </property>
  <property>
    <name>yarn.nodemanager.principal</name>
    <value>yarn/_HOST@XUWANGWEI.TEST</value>
  </property>
  <property>
    <name>yarn.nodemanager.webapp.spnego-principal</name>
    <value>HTTP/_HOST@XUWANGWEI.TEST</value>
  </property>
  <property>
    <name>yarn.nodemanager.keytab</name>
    <value>/etc/hadoop/yarn.keytab</value>
  </property>
  <property>
    <name>yarn.nodemanager.webapp.spnego-keytab-file</name>
    <value>/etc/hadoop/yarn.keytab</value>
  </property>
  <!-- end for kerberos -->
  <property>
    <name>yarn.nodemanager.container-executor.class</name>
    <value>org.apache.hadoop.yarn.server.nodemanager.LinuxContainerExecutor</value>
  </property>
  <property>
    <name>yarn.nodemanager.linux-container-executor.group</name>
    <value>hadoop</value>
  </property>
</configuration>
```

4. 修改配置文件$HADOOP_CONF_DIR/container-executor.cfg
```text
yarn.nodemanager.linux-container-executor.group=hadoop
banned.users=root
#allowed.system.users=
min.user.id=1000
```

5. 修改配置文件$HADOOP_CONF_DIR/mapred-site.xml
```xml
<?xml version="1.0"?>
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
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
  <property>
    <name>mapreduce.jobhistory.address</name>
    <value>machine1.xuwangwei.test:10020</value>
  </property>
  <property>
    <name>mapreduce.jobhistory.webapp.address</name>
    <value>machine1.xuwangwei.test:19888</value>
  </property>
  <!-- start for kerberos -->
  <property>
    <name>mapreduce.jobhistory.principal</name>
    <value>mapred/_HOST@XUWANGWEI.TEST</value>
  </property>
  <property>
    <name>mapreduce.jobhistory.keytab</name>
    <value>/etc/hadoop/mapred.keytab</value>
  </property>
  <!-- end for kerberos -->
</configuration>
```

6. 修改配置文件$HADOOP_CONF_DIR/workers
```text
machine1
machine2
machine3
machine4
machine5
```

7. 创建配置文件 ssl-client.xml
```xml
<?xml version="1.0"?>
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
        <name>ssl.client.truststore.location</name>
        <value>/etc/hadoop/hadoop.p12</value>
        <description>Truststore to be used by clients like distcp. Must be
            specified.
        </description>
    </property>

    <property>
        <name>ssl.client.truststore.password</name>
        <value>xuwangwei3306</value>
        <description>Optional. Default value is "".
        </description>
    </property>

    <property>
        <name>ssl.client.truststore.type</name>
        <value>pkcs12</value>
        <description>Optional. The keystore file format, default value is "jks".
        </description>
    </property>

    <property>
        <name>ssl.client.truststore.reload.interval</name>
        <value>10000</value>
        <description>Truststore reload check interval, in milliseconds.
            Default value is 10000 (10 seconds).
        </description>
    </property>

    <property>
        <name>ssl.client.keystore.location</name>
        <value>/etc/hadoop/hadoop.p12</value>
        <description>Keystore to be used by clients like distcp. Must be
            specified.
        </description>
    </property>

    <property>
        <name>ssl.client.keystore.password</name>
        <value>xuwangwei3306</value>
        <description>Optional. Default value is "".
        </description>
    </property>

    <property>
        <name>ssl.client.keystore.keypassword</name>
        <value>xuwangwei3306</value>
        <description>Optional. Default value is "".
        </description>
    </property>

    <property>
        <name>ssl.client.keystore.type</name>
        <value>pkcs12</value>
        <description>Optional. The keystore file format, default value is "jks".
        </description>
    </property>

</configuration>
```

8. 创建配置文件 ssl-server.xml
```xml
<?xml version="1.0"?>
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
        <name>ssl.server.truststore.location</name>
        <value>/etc/hadoop/hadoop.p12</value>
        <description>Truststore to be used by NN and DN. Must be specified.
        </description>
    </property>

    <property>
        <name>ssl.server.truststore.password</name>
        <value>xuwangwei3306</value>
        <description>Optional. Default value is "".
        </description>
    </property>

    <property>
        <name>ssl.server.truststore.type</name>
        <value>pkcs12</value>
        <description>Optional. The keystore file format, default value is "jks".
        </description>
    </property>

    <property>
        <name>ssl.server.truststore.reload.interval</name>
        <value>10000</value>
        <description>Truststore reload check interval, in milliseconds.
            Default value is 10000 (10 seconds).
        </description>
    </property>

    <property>
        <name>ssl.server.keystore.location</name>
        <value>/etc/hadoop/hadoop.p12</value>
        <description>Keystore to be used by NN and DN. Must be specified.
        </description>
    </property>

    <property>
        <name>ssl.server.keystore.password</name>
        <value>xuwangwei3306</value>
        <description>Must be specified.
        </description>
    </property>

    <property>
        <name>ssl.server.keystore.keypassword</name>
        <value>xuwangwei3306</value>
        <description>Must be specified.
        </description>
    </property>

    <property>
        <name>ssl.server.keystore.type</name>
        <value>pkcs12</value>
        <description>Optional. The keystore file format, default value is "jks".
        </description>
    </property>

    <property>
        <name>ssl.server.exclude.cipher.list</name>
        <value>TLS_ECDHE_RSA_WITH_RC4_128_SHA,SSL_DHE_RSA_EXPORT_WITH_DES40_CBC_SHA,
            SSL_RSA_WITH_DES_CBC_SHA,SSL_DHE_RSA_WITH_DES_CBC_SHA,
            SSL_RSA_EXPORT_WITH_RC4_40_MD5,SSL_RSA_EXPORT_WITH_DES40_CBC_SHA,
            SSL_RSA_WITH_RC4_128_MD5</value>
        <description>Optional. The weak security cipher suites that you want excluded
            from SSL communication.</description>
    </property>

</configuration>
```

9. 创建配置文件 jaas.conf
```text
Client {
       com.sun.security.auth.module.Krb5LoginModule required
       useKeyTab=true
       keyTab="/etc/hadoop/hdfs.keytab"
       storeKey=true
       useTicketCache=false
       principal="hdfs";
};
```

---

## 配置ZooKeeper
> 注意⚠️：下面的命令只需在主机machine1上执行。

1. 创建配置文件$ZOOCFGDIR/zoo.cfg
```bash
mv $ZOOCFGDIR/zoo_sample.cfg $ZOOCFGDIR/zoo.cfg
```

2. 修改配置文件$ZOOCFGDIR/zoo.cfg
```text
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=/var/lib/zookeeper
# the port at which the clients will connect
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1

## Metrics Providers
#
# https://prometheus.io Metrics Exporter
#metricsProvider.className=org.apache.zookeeper.metrics.prometheus.PrometheusMetricsProvider
#metricsProvider.httpPort=7000
#metricsProvider.exportJvmInfo=true

server.1=machine1.xuwangwei.test:2888:3888
server.2=machine2.xuwangwei.test:2888:3888
server.3=machine3.xuwangwei.test:2888:3888
server.4=machine4.xuwangwei.test:2888:3888
server.5=machine5.xuwangwei.test:2888:3888

enforce.auth.enabled=true
enforce.auth.schemes=sasl
authProvider.1=org.apache.zookeeper.server.auth.SASLAuthenticationProvider 
# optional SASL related server-side properties:
# you can instruct ZooKeeper to remove the host from the client principal name during authentication
# (e.g. zk/myhost@EXAMPLE.COM client principal will be authenticated in ZooKeeper as zk@EXAMPLE.COM)
kerberos.removeHostFromPrincipal=true
# you can instruct ZooKeeper to remove the realm from the client principal name during authentication
# (e.g. zk/myhost@EXAMPLE.COM client principal will be authenticated in ZooKeeper as zk/myhost)
kerberos.removeRealmFromPrincipal=true
```

3. 创建配置文件/etc/zookeeper_no_share/jaas.conf，由于该配置文件无法配置 zookeeper/_HOST 这种统一的服务 principal，所以这个配置无法共享，单独放在了一个目录下。每台机器上的此配置都要修改 Server 下 principal。
```text
Server {
       com.sun.security.auth.module.Krb5LoginModule required
       useKeyTab=true
       keyTab="/etc/zookeeper/zookeeper.keytab"
       storeKey=true
       useTicketCache=false
       principal="zookeeper/machine1.xuwangwei.test";
};
Client {
       com.sun.security.auth.module.Krb5LoginModule required
       useKeyTab=true
       keyTab="/etc/zookeeper/zookeeper.keytab"
       storeKey=true
       useTicketCache=false
       principal="zookeeper";
};
```

4. 创建配置文件$ZOOCFGDIR/java.env
```bash
SERVER_JVMFLAGS="${SERVER_JVMFLAGS} -Djava.security.auth.login.config=/etc/zookeeper_no_share/jaas.conf"
CLIENT_JVMFLAGS="${CLIENT_JVMFLAGS} -Djava.security.auth.login.config=/etc/zookeeper_no_share/jaas.conf"
```

5. 在数据目录（即zoo.cfg中配置的dataDir）下创建myid文件和initialize空文件，myid文件内容是本实例在集群中的id。
```bash
su - zookeeper
ssh machine1 'echo 1 > /var/lib/zookeeper/myid;touch /var/lib/zookeeper/initialize'
ssh machine2 'echo 2 > /var/lib/zookeeper/myid;touch /var/lib/zookeeper/initialize'
ssh machine3 'echo 3 > /var/lib/zookeeper/myid;touch /var/lib/zookeeper/initialize'
ssh machine4 'echo 4 > /var/lib/zookeeper/myid;touch /var/lib/zookeeper/initialize'
ssh machine5 'echo 5 > /var/lib/zookeeper/myid;touch /var/lib/zookeeper/initialize'
exit
```

---

## 启动ZooKeeper
> 注意⚠️：下面的命令只需在主机machine1上执行。

```bash
su - zookeeper
ssh machine1 'zkServer.sh start'
ssh machine2 'zkServer.sh start'
ssh machine3 'zkServer.sh start'
ssh machine4 'zkServer.sh start'
ssh machine5 'zkServer.sh start'
exit
```

---

## 启动Hadoop
> 注意⚠️：下面的命令只需在主机machine1上执行。

1. 启动 HDFS
```bash
su - hdfs -c "ssh machine1 'hdfs --daemon start journalnode'"
su - hdfs -c "ssh machine2 'hdfs --daemon start journalnode'"
su - hdfs -c "ssh machine3 'hdfs --daemon start journalnode'"
su - hdfs -c "hdfs namenode -format"
su - hdfs -c "hdfs zkfc -formatZK"
su - hdfs -c "start-dfs.sh"
su - hdfs -c "ssh machine2 'hdfs namenode -bootstrapStandby'"
su - hdfs -c "ssh machine3 'hdfs namenode -bootstrapStandby'"
su - hdfs -c "stop-dfs.sh;start-dfs.sh"
```

2. 修改用户 hdfs 的环境变量
```bash
echo 'export KRB5CCNAME=FILE:/tmp/krb5cc_$UID' >> ~hdfs/.bashrc
```
hadoop 命令不支持 FreeIPA 安装后的`default_ccache_name`配置项（KCM），所以我通过改变 hdfs 用户的环境变量，使得 hdfs 用户使用 kinit 命令进行 Kerberos 身份认证时，使用/tmp/krb5cc_$UID的 credential cache。

3. 使用 hdfs@XUWANGWEI@TEST 帐号进行 Kerberos 认证
```bash
su - hdfs -c "kinit -kt /etc/hadoop/hdfs.keytab hdfs"
```

4. 初始化目录
```bash
su - hdfs -c "hadoop fs -chown hdfs:hadoop /"
su - hdfs -c "hadoop fs -mkdir /tmp /user"
su - hdfs -c "hadoop fs -chmod 1777 /tmp"
```

5. 启动 YARN
```bash
su - yarn -c "start-yarn.sh"
```

6. 启动 MapReduce
```bash
su - mapred -c "mapred --daemon start historyserver"
```
