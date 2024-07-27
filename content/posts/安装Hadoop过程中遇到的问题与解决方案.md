---
title: 安装 Hadoop 过程中遇到的问题与解决方案
date: 2023-12-10
categories: ['安装']
draft: false
description: "本文描述了我在安装Hadoop过程中遇到的几个问题和对应的解决方案。问题一，Shell execution returned exit code: 127. error while loading shared libraries: libcrypto.so.1.1: cannot open shared object file: No such file or directory。问题二，Can't get configured value for yarn.nodemanager.linux-container-executor.group。问题三，(GSS initiate failed) with true cause: (GSS initiate failed)"
---

## 问题一：启动 NodeManager 时报找不到 libcrypto.so.1.1 文件。
原始报错日志：
```text
2023-11-25 16:28:14,058 WARN org.apache.hadoop.yarn.server.nodemanager.containermanager.linux.privileged.PrivilegedOperationExecutor: Shell execution returned exit code: 127. Privil
eged Execution Operation Stderr: 
/opt/hadoop/bin/container-executor: error while loading shared libraries: libcrypto.so.1.1: cannot open shared object file: No such file or directory

Stdout: 
Full command array for failed execution: 
[/opt/hadoop/bin/container-executor, --checksetup]
2023-11-25 16:28:14,061 WARN org.apache.hadoop.yarn.server.nodemanager.LinuxContainerExecutor: Exit code from container executor initialization is : 127
org.apache.hadoop.yarn.server.nodemanager.containermanager.linux.privileged.PrivilegedOperationException: ExitCodeException exitCode=127: /opt/hadoop/bin/container-executor: 
error while loading shared libraries: libcrypto.so.1.1: cannot open shared object file: No such file or directory
```
原因：container-executor 依赖 openssl 1.1 版本的库。

解决方案：

- 如果是 Rocky Linux 9 操作系统，可以通过`dnf install compat-openssl11`来安装 openssl 1.1 版本的库。
- 如果无法通过包管理工具安装，那么可以通过编译安装，如下所示：
```bash
cd /opt
curl -O https://www.openssl.org/source/openssl-1.1.1w.tar.gz
tar -zxf openssl-1.1.1w.tar.gz
cd openssl-1.1.1w
dnf install perl -y
./config && make && make test
make install
ln -s /usr/local/lib64/libcrypto.so.1.1 /usr/lib64/libcrypto.so.1.1
ln -s /usr/local/lib64/libssl.so.1.1 /usr/lib64/libssl.so.1.1
```
## 问题二：启动 NodeManager 时报获取不到配置 yarn.nodemanager.linux-container-executor.group。
原始报错日志：
```text
Can't get configured value for yarn.nodemanager.linux-container-executor.group.
```

解决方案：

1. 先确认 yarn-site.xml 和 container-executor.cfg 两个配置文件中确实设置了 yarn.nodemanager.linux-container-executor.group。

2. 再确认配置文件 container-executor.cfg 的位置。container-executor 只会去 $HADOOP_HOME/etc/hadoop 目录下读取 container-executor.cfg，是写死的。如果你和我一样把 container-executor.cfg 配置文件放在里/etc/hadoop 目录下，那么肯定读取不到，也就出现了上面的错误。你需要自己编译 container-executor，并在编译时传入你想放置 container-executor.cfg 的目录。下面就是编译时传入`-Dcontainer-executor.conf.dir=/etc/hadoop/`参数的例子。
```bash
cd /usr/local/src
curl -O https://dlcdn.apache.org/hadoop/common/hadoop-3.3.6/hadoop-3.3.6-src.tar.gz
tar -zxf hadoop-3.3.6-src.tar.gz
cd hadoop-3.3.6-src/hadoop-yarn-project/hadoop-yarn/hadoop-yarn-server/hadoop-yarn-server-nodemanager/
dnf install cmake gcc gcc-c++ compat-openssl11 -y
mvn package -Pnative -DskipTests=true -Dcontainer-executor.conf.dir=/etc/hadoop/
ll target/native/target/usr/local/bin/container-executor
```

3. 确认可执行文件 container-executor 的权限是不是 6050，属于 root:hadoop。属组 hadoop 不是一定的，取决于 yarn.nodemanager.linux-container-executor.group 配置项的取值。

4. 确认配置文件 container-executor.cfg 的权限是不是 400，属于 root:hadoop。属组 hadoop 不是一定的，取决于 yarn.nodemanager.linux-container-executor.group 配置项的取值。
## 问题三：启动 JournalNode 时报身份认证失败。
原始报错日志：
```text
2023-09-16 20:07:47,980 WARN SecurityLogger.org.apache.hadoop.ipc.Server: Auth failed for machine1.xuwangwei.test:41465 / 192.168.2.61:41465:null (GSS initiate failed) with true cause: (GSS initiate failed)
```
原因：Kerberos 加密类型不匹配（Encryption Type Mismatches）

分析：我使用 tcpdump 在主机 machine1 上进行了抓包。下图是抓取到的 TGS-REQ 请求包，可以看到 hdfs 进程在请求时表明自己支持 4 种加密类型，分别是 18、17、16、23。
![1.jpg](/images/problems_when_install_hadoop/1.jpg)
接着看下图的 TGS-REP 响应包。这个响应包证明 machine1 上的 hdfs 进程获取到了 ticket，这个 ticket 是它与服务端（machine3 上的 hdfs 进程）通信的关键。我们可以看到 ticket 中的加密类型是 20。上一张图片告诉我们 hdfs 支持 18、17、16、23 这 4 种加密类型，并不支持 20，所以当这个 ticket 交给 machine3 上的 hdfs 进程时，它是解不开的。
![2.jpg](/images/problems_when_install_hadoop/2.jpg)
问：为什么会出现这种情况？难道 KDC 不知道服务端支持的加密类型吗？

答：是的，KDC 就是不知道服务端支持的加密类型。在整个 Kerberos 认证的过程中，服务端是不会和 KDC 进行通信的。所以 KDC 只能去数据库中找服务端的密钥有几种加密类型，KDC 会把服务端密钥中加密等级最高的加密类型作为 service ticket 的加密类型。

解决方案：如果你和我一样使用 FreeIPA 来安装 KDC，那么可以通过给 ipa-getkeytab 命令加一个参数 --enctypes=aes256-sha1,aes128-sha1 来指定密钥的加密类型。如果你是直接安装的 KDC，那么可以在创建 principal 时，通过传入参数`-e aes256-sha1,aes128-sha1`来指定密钥的加密类型。

问：为什么我会选择 aes256-sha1(18)、aes128-sha1(17) 作为 hdfs 的密钥加密类型？

答：不同的应用支持的 Kerberos 加密类型是不同的。我们通过抓包已经知道 Hadoop 3.3.6 版本支持 18、17、16、23 四种加密类型了，所以 18、17 作为 hdfs 的密钥加密类型的话，KDC 在挑选 ticket 加密类型时只能选 18、17 中的一种，而这两种是 Hadoop 3.3.6 版本支持的。

问：des3-cbc-sha1(16)、arcfour-hmac-md5(23) 这两种加密类型 Hadoop 也支持，怎么不作为 hdfs 的密钥加密类型？

答：下图是 Kerberos 文档中对它支持的加密类型的描述，arcfour-hmac-md5(23) 已 deprecated，des3-cbc-sha1(16) 没有被列出，说明不支持。
![3.png](/images/problems_when_install_hadoop/3.png)
