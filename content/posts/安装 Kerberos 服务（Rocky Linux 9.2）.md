---
title: 安装Kerberos服务（Rocky Linux 9.2）
date: 2023-10-29
categories: ['安装']
draft: false
---

安装目标：

- [x] 一主一备，共两个KDC实例。
- [x] 主KDC每隔一个小时同步数据库至备KDC。
- [x] 主KDC安装在kdc1.example.com主机上，备KDC安装在kdc2.example.com主机上。

---

前置条件：

- [x] 时钟同步。在本文中，即是kdc1.example.com主机与kdc2.example.com主机需要时钟同步。
- [x] 配置/etc/hosts。

---

## 主KDC的安装与配置

1. 安装KDC软件包
```text
[root@kdc1 ~]# dnf install krb5-server krb5-workstation
```

2. 修改/etc/krb5.conf

修改前
```text
# To opt out of the system crypto-policies configuration of krb5, remove the
# symlink at /etc/krb5.conf.d/crypto-policies which will not be recreated.
includedir /etc/krb5.conf.d/

[logging]
    default = FILE:/var/log/krb5libs.log
    kdc = FILE:/var/log/krb5kdc.log
    admin_server = FILE:/var/log/kadmind.log

[libdefaults]
    dns_lookup_realm = false
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true
    rdns = false
    pkinit_anchors = FILE:/etc/pki/tls/certs/ca-bundle.crt
    spake_preauth_groups = edwards25519
    dns_canonicalize_hostname = fallback
    qualify_shortname = ""
#    default_realm = EXAMPLE.COM
    default_ccache_name = KEYRING:persistent:%{uid}

[realms]
# EXAMPLE.COM = {
#     kdc = kerberos.example.com
#     admin_server = kerberos.example.com
# }

[domain_realm]
# .example.com = EXAMPLE.COM
# example.com = EXAMPLE.COM
```
修改后
```text
# To opt out of the system crypto-policies configuration of krb5, remove the
# symlink at /etc/krb5.conf.d/crypto-policies which will not be recreated.
includedir /etc/krb5.conf.d/

[logging]
    default = FILE:/var/log/krb5libs.log
    kdc = FILE:/var/log/krb5kdc.log
    admin_server = FILE:/var/log/kadmind.log

[libdefaults]
    dns_lookup_realm = false
    dns_lookup_kdc = false
    dns_canonicalize_hostname = false
    rdns = false
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true
    pkinit_anchors = FILE:/etc/pki/tls/certs/ca-bundle.crt
    spake_preauth_groups = edwards25519
    qualify_shortname = ""
    default_realm = EXAMPLE.COM
#    default_ccache_name = KEYRING:persistent:%{uid}

[realms]
 EXAMPLE.COM = {
     kdc = kdc1.example.com
     # 如果你不需要备KDC，删除下一行
     kdc = kdc2.example.com
     admin_server = kdc1.example.com
     # 如果你不需要备KDC，删除下一行
     primary_kdc = kdc1.example.com
 }

[domain_realm]
 .example.com = EXAMPLE.COM
 example.com = EXAMPLE.COM

```

3. /var/kerberos/krb5kdc/kdc.conf配置保持不变
```text
[kdcdefaults]
    kdc_ports = 88
    kdc_tcp_ports = 88
    spake_preauth_kdc_challenge = edwards25519

[realms]
EXAMPLE.COM = {
     master_key_type = aes256-cts-hmac-sha384-192
     acl_file = /var/kerberos/krb5kdc/kadm5.acl
     dict_file = /usr/share/dict/words
     default_principal_flags = +preauth
     admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
     supported_enctypes = aes256-cts-hmac-sha384-192:normal aes128-cts-hmac-sha256-128:normal aes256-cts-hmac-sha1-96:normal aes128-cts-hmac-sha1-96:normal camellia256-cts-cmac:normal camellia128-cts-cmac:normal arcfour-hmac-md5:normal
     # Supported encryption types for FIPS mode:
     #supported_enctypes = aes256-cts-hmac-sha384-192:normal aes128-cts-hmac-sha256-128:normal
}
```

4. 创建数据库并设置主密码
```text
[root@kdc1 ~]# kdb5_util create -s
```

5. /var/kerberos/krb5kdc/kadm5.acl配置保持不变
```text
*/admin@EXAMPLE.COM	*
```

6. 启动服务并设置开机自启
```text
[root@kdc1 ~]# systemctl start krb5kdc.service
[root@kdc1 ~]# systemctl start kadmin.service
[root@kdc1 ~]# systemctl enable krb5kdc.service
[root@kdc1 ~]# systemctl enable kadmin.service
```

7. 设置防火墙
```text
[root@kdc1 ~]# firewall-cmd --add-service=kerberos
[root@kdc1 ~]# firewall-cmd --permanent --add-service=kerberos
[root@kdc1 ~]# firewall-cmd --add-service=kadmin
[root@kdc1 ~]# firewall-cmd --permanent --add-service=kadmin
```
至此，主KDC已经安装完毕。如果需要提高Kerberos服务的可用性，可以安装一个至多个备用服务。安装备用服务的过程写在了下方。

---

## 备KDC的安装与配置

1. 安装KDC软件包
```text
[root@kdc2 ~]# dnf install krb5-server krb5-workstation
```

2. 复制主服务的/etc/krb5.conf文件和/var/kerberos/krb5kdc/kdc.conf文件至备服务的主机
```text
[root@kdc1 ~]# scp /etc/krb5.conf root@kdc2.example.com:/etc/krb5.conf
[root@kdc1 ~]# scp /var/kerberos/krb5kdc/kdc.conf root@kdc2.example.com:/var/kerberos/krb5kdc/kdc.conf
```

3. 创建一个用户principal——管理员tom/admin@EXAMPLE.COM，可以在其他主机使用这个用户连接到KDC（更准确地说是kadmin服务端）并拥有管理员权限。稍后我们就会在备份主机上使用tom/admin@EXAMPLE.COM连接到KDC。创建一个主机principal——host/kdc1.example.com@EXAMPLE.COM，相当于这台主机的身份证号码，在主服务的数据库同步至备服务的数据库时用于证明自己的身份。
```text
[root@kdc1 ~]# kadmin.local -r EXAMPLE.COM
Authenticating as principal root/admin@EXAMPLE.COM with password.
kadmin.local:  addprinc tom/admin
No policy specified for tom/admin@EXAMPLE.COM; defaulting to no policy
Enter password for principal "tom/admin@EXAMPLE.COM": 
Re-enter password for principal "tom/admin@EXAMPLE.COM": 
Principal "tom/admin@EXAMPLE.COM" created.
kadmin.local:  addprinc -randkey host/kdc1.example.com
No policy specified for host/kdc1.example.com@EXAMPLE.COM; defaulting to no policy
Principal "host/kdc1.example.com@EXAMPLE.COM" created.
kadmin.local:  ktadd host/kdc1.example.com
Entry for principal host/kdc1.example.com with kvno 2, encryption type aes256-cts-hmac-sha384-192 added to keytab FILE:/etc/krb5.keytab.
Entry for principal host/kdc1.example.com with kvno 2, encryption type aes128-cts-hmac-sha256-128 added to keytab FILE:/etc/krb5.keytab.
Entry for principal host/kdc1.example.com with kvno 2, encryption type aes256-cts-hmac-sha1-96 added to keytab FILE:/etc/krb5.keytab.
Entry for principal host/kdc1.example.com with kvno 2, encryption type aes128-cts-hmac-sha1-96 added to keytab FILE:/etc/krb5.keytab.
Entry for principal host/kdc1.example.com with kvno 2, encryption type camellia256-cts-cmac added to keytab FILE:/etc/krb5.keytab.
Entry for principal host/kdc1.example.com with kvno 2, encryption type camellia128-cts-cmac added to keytab FILE:/etc/krb5.keytab.
Entry for principal host/kdc1.example.com with kvno 2, encryption type arcfour-hmac added to keytab FILE:/etc/krb5.keytab.
kadmin.local:  quit
```

4. 在备份主机上使用tom/admin@EXAMPLE.COM连接到KDC，创建一个主机principal——host/kdc2.example.com@EXAMPLE.COM，这是备份主机的身份证号码。为什么主服务和备服务要创建各自的principal？因为身份认证是双向的，双方要出示自己的证明，你出示你的身份证，我出示我的身份证。
```text
[root@kdc2 ~]# kadmin -p tom/admin -r EXAMPLE.COM
Authenticating as principal tom/admin with password.
Password for tom/admin@EXAMPLE.COM: 
kadmin:  addprinc -randkey host/kdc2.example.com
No policy specified for host/kdc2.example.com@EXAMPLE.COM; defaulting to no policy
Principal "host/kdc2.example.com@EXAMPLE.COM" created.
kadmin:  ktadd host/kdc2.example.com
Entry for principal host/kdc2.example.com with kvno 2, encryption type aes256-cts-hmac-sha384-192 added to keytab FILE:/etc/krb5.keytab.
Entry for principal host/kdc2.example.com with kvno 2, encryption type aes128-cts-hmac-sha256-128 added to keytab FILE:/etc/krb5.keytab.
Entry for principal host/kdc2.example.com with kvno 2, encryption type aes256-cts-hmac-sha1-96 added to keytab FILE:/etc/krb5.keytab.
Entry for principal host/kdc2.example.com with kvno 2, encryption type aes128-cts-hmac-sha1-96 added to keytab FILE:/etc/krb5.keytab.
Entry for principal host/kdc2.example.com with kvno 2, encryption type camellia256-cts-cmac added to keytab FILE:/etc/krb5.keytab.
Entry for principal host/kdc2.example.com with kvno 2, encryption type camellia128-cts-cmac added to keytab FILE:/etc/krb5.keytab.
Entry for principal host/kdc2.example.com with kvno 2, encryption type arcfour-hmac added to keytab FILE:/etc/krb5.keytab.
kadmin:  quit
```

5. 将主服务的主机principal写入配置文件/var/kerberos/krb5kdc/kpropd.acl，相当于告诉备服务只需认可host/kdc1.example.com@EXAMPLE.COM同步数据库。
```text
[root@kdc2 ~]# echo host/kdc1.example.com@EXAMPLE.COM > /var/kerberos/krb5kdc/kpropd.acl
```

6. 复制数据库主密码
```text
[root@kdc1 ~]# scp /var/kerberos/krb5kdc/.k5.EXAMPLE.COM root@kdc2.example.com:/var/kerberos/krb5kdc/.k5.EXAMPLE.COM
```

7. 启动服务、设置开机自启、设置防火墙
```text
[root@kdc2 ~]# systemctl start kprop.service
[root@kdc2 ~]# systemctl enable kprop.service
[root@kdc2 ~]# firewall-cmd --add-service=kprop
[root@kdc2 ~]# firewall-cmd --permanent --add-service=kprop
```

8. 导出主服务的数据库
```text
[root@kdc1 ~]# kdb5_util dump /var/kerberos/krb5kdc/replica_datatrans
```

9. 同步数据库到备份服务
```text
[root@kdc1 ~]# kprop kdc2.example.com
Database propagation to kdc2.example.com: SUCCEEDED
```

10. 启动服务、设置开机自启、设置防火墙
```text
[root@kdc2 ~]# systemctl start krb5kdc.service
[root@kdc2 ~]# systemctl enable krb5kdc.service
[root@kdc2 ~]# firewall-cmd --add-service=kerberos
[root@kdc2 ~]# firewall-cmd --permanent --add-service=kerberos
```

11. 创建定时任务
```text
[root@kdc1 ~]# vi /etc/cron.hourly/distribute_main_kdc_to_slave
```

12. 定时任务内容
```bash
#!/bin/sh

# Distribute KDC database to slave servers
# Created by Jason Garman for use with MIT Kerberos 5

# Configurables
slavekdcs="kdc2.example.com"
slavedata="/var/kerberos/krb5kdc/replica_datatrans"

success=1

kdb5_util dump ${slavedata}
error=$?
if [ $error -ne 0 ]; then
    echo "Kerberos database dump failed with exit code $error. Exiting."
    exit 1
fi

for kdc in $slavekdcs; do
    kprop -f ${slavedata} ${kdc}
    error=$?
    if [ $error -ne 0 ]; then
        echo "Propagation of database to host ${kdc} failed with exit code $error."
        echo "Continuing with other slave servers."
        success=0
    fi
done

if [ $success -eq 1 ]; then
    echo "Kerberos database successfully replicated to all slaves."
fi

```

13. 修改权限
```text
[root@kdc1 ~]# chmod a+x /etc/cron.hourly/distribute_main_kdc_to_slave
```
如果想查看定时任务的执行日志，可以查看/var/log/cron文件。

---

## KDC各守护进程简介
| 守护进程名称 | 默认监听端口号 | 职责 |
| --- | --- | --- |
| krb5kdc | 88 | 身份认证。 |
| kadmind | 464 | 处理修改密码请求。 |
|  | 749 | 处理对principal的增删改查、导出等请求。 |
| kpropd | 754 | 同步数据库。备服务才需要启动，主服务不需要启动。 |

