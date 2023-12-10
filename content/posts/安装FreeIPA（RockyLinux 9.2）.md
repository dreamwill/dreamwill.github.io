---
title: 安装FreeIPA（Rocky Linux 9.2）
date: 2023-08-27
categories: ['安装']
draft: false
---

安装目标：

- [x] 安装一个FreeIPA服务端实例，该实例所在主机 IP 地址为 192.168.2.60，对应主机 ipa-server.xuwangwei.test。
- [x] 安装一个FreeIPA客户端实例，该实例所在主机 IP 地址为 192.168.2.61，对应主机 machine1.xuwangwei.test。

---

前置条件：

- [x] 无。

---

## 安装服务端

1. 设置主机名
```bash
hostnamectl hostname ipa-server.xuwangwei.test
```

2. 修改配置/etc/hosts

```text
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.2.60 ipa-server.xuwangwei.test ipa-server
```

3. 安装
```bash
dnf install ipa-server ipa-server-dns rng-tools
```

4. 启动随机数生成器服务rngd.service
```bash
systemctl start rngd
```

5. 设置服务端
```bash
ipa-server-install --realm=XUWANGWEI.TEST --domain=xuwangwei.test \
--hostname=ipa-server.xuwangwei.test --ip-address=192.168.2.60 \
--ntp-pool=time.apple.com --mkhomedir \
--setup-dns --no-forwarders --reverse-zone=2.168.192.in-addr.arpa --auto-reverse --netbios-name=XUWANGWEI \
--ds-password='xuwangwei3306' --admin-password='xuwangwei3306' --unattended
```
下面是输出：
```text

Checking DNS domain 2.168.192.in-addr.arpa, please wait ...

The log file for this installation can be found in /var/log/ipaserver-install.log
==============================================================================
This program will set up the IPA Server.
Version 4.10.2

This includes:
  * Configure a stand-alone CA (dogtag) for certificate management
  * Configure the NTP client (chronyd)
  * Create and configure an instance of Directory Server
  * Create and configure a Kerberos Key Distribution Center (KDC)
  * Configure Apache (httpd)
  * Configure DNS (bind)
  * Configure SID generation
  * Configure the KDC to enable PKINIT

Warning: skipping DNS resolution of host ipa-server.xuwangwei.test
Checking DNS domain xuwangwei.test., please wait ...
Checking DNS domain 2.168.192.in-addr.arpa, please wait ...
Checking DNS domain 2.168.192.in-addr.arpa, please wait ...
Using reverse zone(s) 2.168.192.in-addr.arpa.

The IPA Master Server will be configured with:
Hostname:       ipa-server.xuwangwei.test
IP address(es): 192.168.2.60
Domain name:    xuwangwei.test
Realm name:     XUWANGWEI.TEST

The CA will be configured with:
Subject DN:   CN=Certificate Authority,O=XUWANGWEI.TEST
Subject base: O=XUWANGWEI.TEST
Chaining:     self-signed

BIND DNS server will be configured to serve IPA domain with:
Forwarders:       No forwarders
Forward policy:   only
Reverse zone(s):  2.168.192.in-addr.arpa.

NTP pool:	time.apple.com
Disabled p11-kit-proxy
Synchronizing time
Configuration of chrony was changed by installer.
Attempting to sync time with chronyc.
Time synchronization was successful.
Configuring directory server (dirsrv). Estimated time: 30 seconds
  [1/43]: creating directory server instance
Validate installation settings ...
Create file system structures ...
selinux is disabled, will not relabel ports or files.
Create database backend: dc=xuwangwei,dc=test ...
Perform post-installation tasks ...
  [2/43]: tune ldbm plugin
  [3/43]: adding default schema
  [4/43]: enabling memberof plugin
  [5/43]: enabling winsync plugin
  [6/43]: configure password logging
  [7/43]: configuring replication version plugin
  [8/43]: enabling IPA enrollment plugin
  [9/43]: configuring uniqueness plugin
  [10/43]: configuring uuid plugin
  [11/43]: configuring modrdn plugin
  [12/43]: configuring DNS plugin
  [13/43]: enabling entryUSN plugin
  [14/43]: configuring lockout plugin
  [15/43]: configuring graceperiod plugin
  [16/43]: configuring topology plugin
  [17/43]: creating indices
  [18/43]: enabling referential integrity plugin
  [19/43]: configuring certmap.conf
  [20/43]: configure new location for managed entries
  [21/43]: configure dirsrv ccache and keytab
  [22/43]: enabling SASL mapping fallback
  [23/43]: restarting directory server
  [24/43]: adding sasl mappings to the directory
  [25/43]: adding default layout
  [26/43]: adding delegation layout
  [27/43]: creating container for managed entries
  [28/43]: configuring user private groups
  [29/43]: configuring netgroups from hostgroups
  [30/43]: creating default Sudo bind user
  [31/43]: creating default Auto Member layout
  [32/43]: adding range check plugin
  [33/43]: creating default HBAC rule allow_all
  [34/43]: adding entries for topology management
  [35/43]: initializing group membership
  [36/43]: adding master entry
  [37/43]: initializing domain level
  [38/43]: configuring Posix uid/gid generation
  [39/43]: adding replication acis
  [40/43]: activating sidgen plugin
  [41/43]: activating extdom plugin
  [42/43]: configuring directory to start on boot
  [43/43]: restarting directory server
Done configuring directory server (dirsrv).
Configuring Kerberos KDC (krb5kdc)
  [1/11]: adding kerberos container to the directory
  [2/11]: configuring KDC
  [3/11]: initialize kerberos container
  [4/11]: adding default ACIs
  [5/11]: creating a keytab for the directory
  [6/11]: creating a keytab for the machine
  [7/11]: adding the password extension to the directory
  [8/11]: creating anonymous principal
  [9/11]: starting the KDC
  [10/11]: configuring KDC to start on boot
  [11/11]: enable PAC ticket signature support
Done configuring Kerberos KDC (krb5kdc).
Configuring kadmin
  [1/2]: starting kadmin 
  [2/2]: configuring kadmin to start on boot
Done configuring kadmin.
Configuring ipa-custodia
  [1/5]: Making sure custodia container exists
  [2/5]: Generating ipa-custodia config file
  [3/5]: Generating ipa-custodia keys
  [4/5]: starting ipa-custodia 
  [5/5]: configuring ipa-custodia to start on boot
Done configuring ipa-custodia.
Configuring certificate server (pki-tomcatd). Estimated time: 3 minutes
  [1/30]: configuring certificate server instance
  [2/30]: stopping certificate server instance to update CS.cfg
  [3/30]: backing up CS.cfg
  [4/30]: Add ipa-pki-wait-running
  [5/30]: secure AJP connector
  [6/30]: reindex attributes
  [7/30]: exporting Dogtag certificate store pin
  [8/30]: disabling nonces
  [9/30]: set up CRL publishing
  [10/30]: enable PKIX certificate path discovery and validation
  [11/30]: authorizing RA to modify profiles
  [12/30]: authorizing RA to manage lightweight CAs
  [13/30]: Ensure lightweight CAs container exists
  [14/30]: Ensuring backward compatibility
  [15/30]: starting certificate server instance
  [16/30]: configure certmonger for renewals
  [17/30]: requesting RA certificate from CA
  [18/30]: publishing the CA certificate
  [19/30]: adding RA agent as a trusted user
  [20/30]: configure certificate renewals
  [21/30]: Configure HTTP to proxy connections
  [22/30]: updating IPA configuration
  [23/30]: enabling CA instance
  [24/30]: importing IPA certificate profiles
  [25/30]: migrating certificate profiles to LDAP
  [26/30]: adding default CA ACL
  [27/30]: adding 'ipa' CA entry
  [28/30]: Recording random serial number state
  [29/30]: configuring certmonger renewal for lightweight CAs
  [30/30]: deploying ACME service
Done configuring certificate server (pki-tomcatd).
Configuring directory server (dirsrv)
  [1/3]: configuring TLS for DS instance
  [2/3]: adding CA certificate entry
  [3/3]: restarting directory server
Done configuring directory server (dirsrv).
Configuring ipa-otpd
  [1/2]: starting ipa-otpd 
  [2/2]: configuring ipa-otpd to start on boot
Done configuring ipa-otpd.
Configuring the web interface (httpd)
  [1/22]: stopping httpd
  [2/22]: backing up ssl.conf
  [3/22]: disabling nss.conf
  [4/22]: configuring mod_ssl certificate paths
  [5/22]: setting mod_ssl protocol list
  [6/22]: configuring mod_ssl log directory
  [7/22]: disabling mod_ssl OCSP
  [8/22]: adding URL rewriting rules
  [9/22]: configuring httpd
Nothing to do for configure_httpd_wsgi_conf
  [10/22]: setting up httpd keytab
  [11/22]: configuring Gssproxy
  [12/22]: setting up ssl
  [13/22]: configure certmonger for renewals
  [14/22]: publish CA cert
  [15/22]: clean up any existing httpd ccaches
  [16/22]: enable ccache sweep
  [17/22]: configuring SELinux for httpd
  [18/22]: create KDC proxy config
  [19/22]: enable KDC proxy
  [20/22]: starting httpd
  [21/22]: configuring httpd to start on boot
  [22/22]: enabling oddjobd
Done configuring the web interface (httpd).
Configuring Kerberos KDC (krb5kdc)
  [1/1]: installing X509 Certificate for PKINIT
Done configuring Kerberos KDC (krb5kdc).
Applying LDAP updates
Upgrading IPA:. Estimated time: 1 minute 30 seconds
  [1/10]: stopping directory server
  [2/10]: saving configuration
  [3/10]: disabling listeners
  [4/10]: enabling DS global lock
  [5/10]: disabling Schema Compat
  [6/10]: starting directory server
  [7/10]: upgrading server
  [8/10]: stopping directory server
  [9/10]: restoring configuration
  [10/10]: starting directory server
Done.
Restarting the KDC
dnssec-validation yes
Configuring DNS (named)
  [1/13]: generating rndc key file
  [2/13]: adding DNS container
  [3/13]: setting up our zone
  [4/13]: setting up reverse zone
  [5/13]: setting up our own record
  [6/13]: setting up records for other masters
  [7/13]: adding NS record to the zones
  [8/13]: setting up kerberos principal
  [9/13]: setting up LDAPI autobind
  [10/13]: setting up named.conf
created new /etc/named.conf
created named user config '/etc/named/ipa-ext.conf'
created named user config '/etc/named/ipa-options-ext.conf'
created named user config '/etc/named/ipa-logging-ext.conf'
  [11/13]: setting up server configuration
  [12/13]: configuring named to start on boot
  [13/13]: changing resolv.conf to point to ourselves
Done configuring DNS (named).
Restarting the web server to pick up resolv.conf changes
Configuring DNS key synchronization service (ipa-dnskeysyncd)
  [1/7]: checking status
  [2/7]: setting up bind-dyndb-ldap working directory
  [3/7]: setting up kerberos principal
  [4/7]: setting up SoftHSM
  [5/7]: adding DNSSEC containers
  [6/7]: creating replica keys
  [7/7]: configuring ipa-dnskeysyncd to start on boot
Done configuring DNS key synchronization service (ipa-dnskeysyncd).
Restarting ipa-dnskeysyncd
Restarting named
Updating DNS system records
Configuring SID generation
  [1/8]: adding RID bases
  [2/8]: creating samba domain object
  [3/8]: adding admin(group) SIDs
  [4/8]: updating Kerberos config
'dns_lookup_kdc' already set to 'true', nothing to do.
  [5/8]: activating sidgen task
  [6/8]: restarting Directory Server to take MS PAC and LDAP plugins changes into account
  [7/8]: adding fallback group
  [8/8]: adding SIDs to existing users and groups
This step may take considerable amount of time, please wait..
Done.
Configuring client side components
This program will set up IPA client.
Version 4.10.2

Using existing certificate '/etc/ipa/ca.crt'.
Client hostname: ipa-server.xuwangwei.test
Realm: XUWANGWEI.TEST
DNS Domain: xuwangwei.test
IPA Server: ipa-server.xuwangwei.test
BaseDN: dc=xuwangwei,dc=test

Configured /etc/sssd/sssd.conf
Systemwide CA database updated.
Adding SSH public key from /etc/ssh/ssh_host_ecdsa_key.pub
Adding SSH public key from /etc/ssh/ssh_host_ed25519_key.pub
Adding SSH public key from /etc/ssh/ssh_host_rsa_key.pub
SSSD enabled
Configured /etc/openldap/ldap.conf
Configured /etc/ssh/ssh_config
Configured /etc/ssh/sshd_config.d/04-ipa.conf
Configuring xuwangwei.test as NIS domain.
Client configuration complete.
The ipa-client-install command was successful

==============================================================================
Setup complete

Next steps:
	1. You must make sure these network ports are open:
		TCP Ports:
		  * 80, 443: HTTP/HTTPS
		  * 389, 636: LDAP/LDAPS
		  * 88, 464: kerberos
		  * 53: bind
		UDP Ports:
		  * 88, 464: kerberos
		  * 53: bind
		  * 123: ntp

	2. You can now obtain a kerberos ticket using the command: 'kinit admin'
	   This ticket will allow you to use the IPA tools (e.g., ipa user-add)
	   and the web user interface.

Be sure to back up the CA certificates stored in /root/cacert.p12
These files are required to create replicas. The password for these
files is the Directory Manager password
The ipa-server-install command was successful
```

6. 允许PTR同步
```console
[root@ipa-server ~]# kinit admin
Password for admin@XUWANGWEI.TEST: 
[root@ipa-server ~]# ipa dnsconfig-mod --allow-sync-ptr=true
  允许PTR同步: True
  IPA DNS服务器: ipa-server.xuwangwei.test
```

7. 设置防火墙
```bash
firewall-cmd --add-service={freeipa-4,dns}
firewall-cmd --add-service={freeipa-4,dns} --permanent
```
至此，服务端已安装完毕。

---

## 进入Web管理平台（可选）

1. 用浏览器打开https://ipa-server.xuwangwei.test

![截屏2023-08-27 02.49.37.png](/images/截屏2023-08-27-02.49.37.png)

2. 输入用户名admin和密码后进入管理页面

![20230903171821.jpg](/images/20230903171821.jpg)

---

## 安装客户端

1. 设置主机名
```bash
hostnamectl hostname machine1.xuwangwei.test
```

2. 修改配置/etc/hosts

```text
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.2.61 machine1.xuwangwei.test machine1
```

3. 安装
```bash
dnf install ipa-client
```

4. 设置DNS
```bash
nmcli c modify ens160 ipv4.dns 192.168.2.60
nmcli c up id ens160
```

5. 设置客户端
```bash
ipa-client-install --ntp-pool=time.apple.com --mkhomedir \
--principal admin --password='xuwangwei3306' --unattended
```
下面是设置过程中的输出。客户端的设置过程相较服务端的设置简单很多，速度也快很多。
```text
This program will set up IPA client.
Version 4.10.2

Discovery was successful!
Client hostname: machine1.xuwangwei.test
Realm: XUWANGWEI.TEST
DNS Domain: xuwangwei.test
IPA Server: ipa-server.xuwangwei.test
BaseDN: dc=xuwangwei,dc=test
NTP pool: time.apple.com

Synchronizing time
Configuration of chrony was changed by installer.
Attempting to sync time with chronyc.
Time synchronization was successful.
Successfully retrieved CA cert
    Subject:     CN=Certificate Authority,O=XUWANGWEI.TEST
    Issuer:      CN=Certificate Authority,O=XUWANGWEI.TEST
    Valid From:  2023-12-10 01:11:36
    Valid Until: 2043-12-10 01:11:36

Enrolled in IPA realm XUWANGWEI.TEST
Created /etc/ipa/default.conf
Configured /etc/sssd/sssd.conf
Systemwide CA database updated.
Hostname (machine1.xuwangwei.test) does not have A/AAAA record.
Missing reverse record(s) for address(es): fdb4:1b:7835:4c67:20c:29ff:fe7c:a184.
Adding SSH public key from /etc/ssh/ssh_host_ecdsa_key.pub
Adding SSH public key from /etc/ssh/ssh_host_ed25519_key.pub
Adding SSH public key from /etc/ssh/ssh_host_rsa_key.pub
SSSD enabled
Configured /etc/openldap/ldap.conf
Configured /etc/ssh/ssh_config
Configured /etc/ssh/sshd_config.d/04-ipa.conf
Configuring xuwangwei.test as NIS domain.
Configured /etc/krb5.conf for IPA realm XUWANGWEI.TEST
Client configuration complete.
The ipa-client-install command was successful
```

6. 查看Web管理页面

![截屏2023-12-10-10.20.52.png](/images/截屏2023-12-10-10.20.52.png)
从上图中可以看到客户端已经成功注册。至此，客户端已经安装完毕。
