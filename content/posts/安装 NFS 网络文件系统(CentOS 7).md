---
title: 安装 NFS 网络文件系统(CentOS 7)
date: 2024-01-30
categories: ['安装']
draft: false
---

安装目标：

- [x] 安装一个 NFS(Network File System) 服务端实例，该实例所在主机的 IP 地址为 192.168.2.70，对应主机名 nfs.xuwangwei.test。
- [x] 共享 nfs.xuwangwei.test 主机的/home 目录，使用 client1.xuwangwei.test 主机进行挂载测试。

---

前置条件：

- [x] 无。

---

## 安装服务端
> 注意⚠️：下面的命令是在主机 nfs.xuwangwei.test 上执行的。

1. 安装
```console
[root@nfs ~]# yum install nfs-utils
```

2. 设置防火墙
```console
[root@nfs ~]# firewall-cmd --add-service={rpc-bind,mountd,nfs}
[root@nfs ~]# firewall-cmd --add-service={rpc-bind,mountd,nfs} --permanent
```

3. 启动服务
```console
[root@nfs ~]# systemctl start rpcbind nfs-mountd nfs
```

4. 设置开机自启动
```console
[root@nfs ~]# systemctl enable rpcbind nfs-mountd nfs
```

5. 修改配置文件/etc/exports。假设我想共享/home 目录，使得 192.168.2.0/24 网络下的所有主机都有权限使用该目录，且该目录允许读写。
```console
/home 192.168.2.0/24(rw,sync,no_root_squash)
```

6. 使配置文件生效
```console
[root@nfs ~]# exportfs -rv
exporting 192.168.2.0/24:/home
```

---

## 测试配置是否正确
> 注意⚠️：下面的命令有在主机 nfs.xuwangwei.test 上执行的，也有在主机 client1.xuwangwei.test 上执行的。请根据控制台的提示分辨。

1. 在主机 nfs.xuwangwei.test 上创建用户和用户组用于测试
```console
[root@nfs ~]# groupadd --gid 2000 xuwangwei
[root@nfs ~]# useradd --create-home --gid 2000 --uid 2000 xuwangwei
```

2. 在/home/xuwangwei 目录下新建一个文本文件 1.txt，内容是 hello
```console
[root@nfs ~]# su - xuwangwei -c 'echo "hello" >> /home/xuwangwei/1.txt'
```

3. 查看 1.txt 是否新建成功，是否写入了 hello
```console
[root@nfs ~]# ls -l /home/xuwangwei/1.txt
-rw-rw-r--. 1 xuwangwei xuwangwei 6 1月  30 17:45 /home/xuwangwei/1.txt
[root@nfs ~]# cat /home/xuwangwei/1.txt
hello
```

4. 在主机 client1.xuwangwei.test 上创建同样的用户和用户组
```console
[root@client1 ~]# groupadd --gid 2000 xuwangwei
[root@client1 ~]# useradd --create-home --gid 2000 --uid 2000 xuwangwei
```

5. 安装
```console
[root@client1 ~]# yum install nfs-utils
```

6. 挂载目录
```console
[root@client1 ~]# mount -v -t nfs4 192.168.2.70:/home /home
mount.nfs4: timeout set for Tue Jan 30 18:01:48 2024
mount.nfs4: trying text-based options 'vers=4.1,addr=192.168.2.70,clientaddr=192.168.2.71'
```

7. 查看是否挂载成功
```console
[root@client1 ~]# mount |grep 192.168.2.70
192.168.2.70:/home on /home type nfs4 (rw,relatime,vers=4.1,rsize=1048576,wsize=1048576,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.2.71,local_lock=none,addr=192.168.2.70)
```

8. 查看 1.txt 是否存在，且内容是否正确
```console
[root@client1 ~]# ls -l /home/xuwangwei
总用量 4
-rw-rw-r--. 1 xuwangwei xuwangwei 6 1月  30 17:45 1.txt
[root@client1 ~]# cat /home/xuwangwei/1.txt 
hello
```
