---
title: "使用 Ansible 一键安装 FreeIPA(Rocky Linux 9)"
date: 2024-12-07
categories: ['安装']
draft: false
---

安装目标：

- [x] 安装一个 FreeIPA 服务端实例，该实例所在主机 IP 地址为 192.168.2.60，对应主机名 ipa-server.xuwangwei.test。
- [x] 安装两个 FreeIPA 客户端实例，两台主机 IP 地址分别为 192.168.2.61、192.168.2.62，对应主机名 machine1.xuwangwei.test、machine2.xuwangwei.test。

---

前置条件：

- [x] Ansible 管理节点[^1]需要能够通过 `ssh` 免密登录到目标节点[^2]。
- [x] 登录目标节点的账号需要能够免密执行 `sudo` 命令。

---

## 安装服务端
1. 安装 Ansible

```bash
sudo dnf install ansible-core -y
```

2. 安装 Ansible collection

```bash
ansible-galaxy collection install freeipa.ansible_freeipa:1.14.1
ansible-galaxy collection install ansible.posix:1.6.1
```

3. 创建目录，存放 inventory 和 playbook

```bash
mkdir deploy_freeipa_using_ansible && cd deploy_freeipa_using_ansible
```

4. 创建 inventory

文件内容：

```ini
[ipaserver]
ipa-server.xuwangwei.test ansible_host=192.168.2.60

[ipaserver:vars]
ipadm_password=xuwangwei3306
ipaadmin_password=xuwangwei3306
ipaserver_ip_addresses=192.168.2.60
ipaserver_domain=xuwangwei.test
ipaserver_realm=XUWANGWEI.TEST
ipaserver_hostname=ipa-server.xuwangwei.test
ipaserver_no_host_dns=yes
ipaserver_setup_dns=yes
ipaserver_reverse_zones=2.168.192.in-addr.arpa
ipaserver_auto_reverse=yes
ipaserver_auto_forwarders=yes
ipaserver_setup_firewalld=yes
ipaserver_firewalld_zone=public

[ipaclients]
machine1.xuwangwei.test ansible_host=192.168.2.61
machine2.xuwangwei.test ansible_host=192.168.2.62

[ipaclients:vars]
ipaclient_domain=xuwangwei.test
ipaclient_mkhomedir=yes
ipaclient_ntp_servers=192.168.2.60
ipaclient_configure_dns_resolver=yes
ipaclient_dns_servers=192.168.2.60
ipaadmin_password=xuwangwei3306
```

5. 创建 deploy_freeipa.yml

文件内容：

```yaml
---
- name: Configure IPA server
  hosts: ipaserver
  become: true
  pre_tasks:
    - name: Set the hostname
      ansible.builtin.hostname:
        name: "{{ inventory_hostname }}"

    - name: Update /etc/hosts with server details
      ansible.builtin.lineinfile:
        path: /etc/hosts
        line: "{{ ansible_host }} {{ inventory_hostname }} {{ inventory_hostname_short }}"
        state: present
        create: true

  roles:
    - role: freeipa.ansible_freeipa.ipaserver
      state: present

  post_tasks:
    - name: Ensure allow_sync_ptr is yes
      freeipa.ansible_freeipa.ipadnsconfig:
        ipaadmin_password: xuwangwei3306
        allow_sync_ptr: yes

    - name: Add FreeIPA service to firewalld (temporary and permanent)
      ansible.posix.firewalld:
        service: freeipa-4
        state: enabled
        permanent: true
        immediate: true

    - name: Add DNS service to firewalld (temporary and permanent)
      ansible.posix.firewalld:
        service: dns
        state: enabled
        permanent: true
        immediate: true

    - name: Add NTP service to firewalld (temporary and permanent)
      ansible.posix.firewalld:
        service: ntp
        state: enabled
        permanent: true
        immediate: true

    - name: Ensure /etc/chrony.conf contains allow for LAN
      ansible.builtin.lineinfile:
        path: /etc/chrony.conf
        regexp: '^allow '
        line: 'allow 192.168.2.0/24'
        state: present

    - name: Restart chronyd service to apply changes
      ansible.builtin.service:
        name: chronyd
        state: restarted
        enabled: true

- name: Configure IPA clients
  hosts: ipaclients
  become: true
  pre_tasks:
    - name: Set the hostname
      ansible.builtin.hostname:
        name: "{{ inventory_hostname }}"

    - name: Update /etc/hosts with client details
      ansible.builtin.lineinfile:
        path: /etc/hosts
        line: "{{ ansible_host }} {{ inventory_hostname }} {{ inventory_hostname_short }}"
        state: present
        create: true

  roles:
    - role: freeipa.ansible_freeipa.ipaclient
      state: present

```

6. 一键安装

```bash
ansible-playbook -i inventory deploy_freeipa.yml
```

下面是输出：

```plain

PLAY [Configure IPA server] ********************************************************************************************

TASK [Gathering Facts] *************************************************************************************************
ok: [ipa-server.xuwangwei.test]

TASK [Set the hostname] ************************************************************************************************
changed: [ipa-server.xuwangwei.test]

TASK [Update /etc/hosts with server details] ***************************************************************************
changed: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Import variables specific to distribution] ***********************************
ok: [ipa-server.xuwangwei.test] => (item=/home/vagrant/.ansible/collections/ansible_collections/freeipa/ansible_freeipa/roles/ipaserver/vars/default.yml)

TASK [freeipa.ansible_freeipa.ipaserver : Install IPA server] **********************************************************
included: /home/vagrant/.ansible/collections/ansible_collections/freeipa/ansible_freeipa/roles/ipaserver/tasks/install.yml for ipa-server.xuwangwei.test

TASK [freeipa.ansible_freeipa.ipaserver : Install - Ensure that IPA server packages are installed] *********************
changed: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Install - Ensure that IPA server packages for dns are installed] *************
changed: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Install - Ensure that IPA server packages for adtrust are installed] *********
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Install - Ensure that firewall packages installed] ***************************
ok: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Firewalld service - Ensure that firewalld is running] ************************
changed: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Firewalld - Verify runtime zone "public"] ************************************
changed: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Firewalld - Verify permanent zone "public"] **********************************
changed: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Copy external certs] *********************************************************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Install - Server installation test] ******************************************
ok: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Install - Master password creation] ******************************************
changed: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Install - Use new master password] *******************************************
ok: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Use user defined master password, if provided] *******************************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Install - Server preparation] ************************************************
changed: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Install - Setup NTP] *********************************************************
changed: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Install - Setup DS] **********************************************************
changed: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Install - Setup KRB] *********************************************************
changed: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Install - Setup CA] **********************************************************
changed: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Copy /root/ipa.csr to "ipa-server.xuwangwei.test-ipa.csr"] *******************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Install - Setup otpd] ********************************************************
changed: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Install - Setup HTTP] ********************************************************
changed: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Install - Setup KRA] *********************************************************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Install - Setup DNS] *********************************************************
changed: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Install - Setup ADTRUST] *****************************************************
changed: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Install - Set DS password] ***************************************************
changed: [ipa-server.xuwangwei.test]

TASK [Install - Setup client] ******************************************************************************************

TASK [freeipa.ansible_freeipa.ipaclient : Import variables specific to distribution] ***********************************
ok: [ipa-server.xuwangwei.test] => (item=/home/vagrant/.ansible/collections/ansible_collections/freeipa/ansible_freeipa/roles/ipaclient/vars/default.yml)

TASK [freeipa.ansible_freeipa.ipaclient : Install IPA client] **********************************************************
included: /home/vagrant/.ansible/collections/ansible_collections/freeipa/ansible_freeipa/roles/ipaclient/tasks/install.yml for ipa-server.xuwangwei.test

TASK [freeipa.ansible_freeipa.ipaclient : Install - Ensure that IPA client packages are installed] *********************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Set ipaclient_servers] *********************************************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Set ipaclient_servers from cluster inventory] **********************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Check that either password or keytab is set] ***********************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Set default principal if no keytab is given] ***********************
ok: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Fail on missing ipaclient_domain and ipaserver_domain] *************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Fail on missing ipaclient_servers] *********************************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Configure DNS resolver] ********************************************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - IPA client test] ***************************************************
ok: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Cleanup leftover ccache] *******************************************
ok: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Configure NTP] *****************************************************
ok: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Make sure One-Time Password is enabled if it's already defined] ****
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Disable One-Time Password for on_master] ***************************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Test if IPA client has working krb5.keytab] ************************
ok: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Disable One-Time Password for client with working krb5.keytab] *****
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Keytab or password is required for getting otp] ********************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Create temporary file for keytab] **********************************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Copy keytab to server temporary file] ******************************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Get One-Time Password for client enrollment] ***********************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Store the previously obtained OTP] *********************************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Remove keytab temporary file] **************************************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Store predefined OTP in admin_password] **************************************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Check if principal and keytab are set] *****************************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Check if one of password or keytabs are set] ***********************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - From host keytab, purge XUWANGWEI.TEST] ****************************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Backup and set hostname] *******************************************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Create temporary krb5 configuration] *******************************
ok: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Join IPA] **********************************************************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : The krb5 configuration is not correct] ***************************************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : IPA test failed] *************************************************************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Fail due to missing ca.crt file] *********************************************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Configure IPA default.conf] ****************************************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Configure SSSD] ****************************************************
changed: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - IPA API calls for remaining enrollment parts] **********************
changed: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Fix IPA ca] ********************************************************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Create IPA NSS database] *******************************************
changed: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Configure SSH and SSHD] ********************************************
changed: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Configure automount] ***********************************************
ok: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Configure firefox] *************************************************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Configure NIS] *****************************************************
changed: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Remove temporary krb5.conf] **************************************************
changed: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Configure krb5 for IPA realm] **************************************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Configure certmonger] **********************************************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Restore original admin password if overwritten by OTP] *************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Cleanup leftover ccache] *****************************************************
ok: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Remove temporary krb5.conf] **************************************************
ok: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Remove temporary krb5.conf backup] *******************************************
changed: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Uninstall IPA client] ********************************************************
skipping: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Install - Enable IPA] ********************************************************
changed: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Install - Configure firewalld] ***********************************************
changed: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Install - Configure firewalld runtime] ***************************************
changed: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Install - Cleanup root IPA cache] ********************************************
ok: [ipa-server.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaserver : Cleanup temporary files] *****************************************************
ok: [ipa-server.xuwangwei.test] => (item=/etc/ipa/.tmp_pkcs12_dirsrv)
ok: [ipa-server.xuwangwei.test] => (item=/etc/ipa/.tmp_pkcs12_http)
ok: [ipa-server.xuwangwei.test] => (item=/etc/ipa/.tmp_pkcs12_pkinit)

TASK [freeipa.ansible_freeipa.ipaserver : Uninstall IPA server] ********************************************************
skipping: [ipa-server.xuwangwei.test]

TASK [Ensure allow_sync_ptr is yes] ************************************************************************************
changed: [ipa-server.xuwangwei.test]

TASK [Add FreeIPA service to firewalld (temporary and permanent)] ******************************************************
changed: [ipa-server.xuwangwei.test]

TASK [Add DNS service to firewalld (temporary and permanent)] **********************************************************
ok: [ipa-server.xuwangwei.test]

TASK [Add NTP service to firewalld (temporary and permanent)] **********************************************************
ok: [ipa-server.xuwangwei.test]

TASK [Ensure /etc/chrony.conf contains allow for LAN] ******************************************************************
changed: [ipa-server.xuwangwei.test]

TASK [Restart chronyd service to apply changes] ************************************************************************
changed: [ipa-server.xuwangwei.test]

PLAY [Configure IPA clients] *******************************************************************************************

TASK [Gathering Facts] *************************************************************************************************
ok: [machine1.xuwangwei.test]
ok: [machine2.xuwangwei.test]

TASK [Set the hostname] ************************************************************************************************
changed: [machine2.xuwangwei.test]
changed: [machine1.xuwangwei.test]

TASK [Update /etc/hosts with client details] ***************************************************************************
changed: [machine1.xuwangwei.test]
changed: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Import variables specific to distribution] ***********************************
ok: [machine1.xuwangwei.test] => (item=/home/vagrant/.ansible/collections/ansible_collections/freeipa/ansible_freeipa/roles/ipaclient/vars/default.yml)
ok: [machine2.xuwangwei.test] => (item=/home/vagrant/.ansible/collections/ansible_collections/freeipa/ansible_freeipa/roles/ipaclient/vars/default.yml)

TASK [freeipa.ansible_freeipa.ipaclient : Install IPA client] **********************************************************
included: /home/vagrant/.ansible/collections/ansible_collections/freeipa/ansible_freeipa/roles/ipaclient/tasks/install.yml for machine1.xuwangwei.test, machine2.xuwangwei.test

TASK [freeipa.ansible_freeipa.ipaclient : Install - Ensure that IPA client packages are installed] *********************
changed: [machine1.xuwangwei.test]
changed: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Set ipaclient_servers] *********************************************
skipping: [machine1.xuwangwei.test]
skipping: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Set ipaclient_servers from cluster inventory] **********************
skipping: [machine1.xuwangwei.test]
skipping: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Check that either password or keytab is set] ***********************
skipping: [machine1.xuwangwei.test]
skipping: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Set default principal if no keytab is given] ***********************
ok: [machine1.xuwangwei.test]
ok: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Fail on missing ipaclient_domain and ipaserver_domain] *************
skipping: [machine1.xuwangwei.test]
skipping: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Fail on missing ipaclient_servers] *********************************
skipping: [machine1.xuwangwei.test]
skipping: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Configure DNS resolver] ********************************************
changed: [machine1.xuwangwei.test]
changed: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - IPA client test] ***************************************************
ok: [machine1.xuwangwei.test]
ok: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Cleanup leftover ccache] *******************************************
ok: [machine1.xuwangwei.test]
ok: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Configure NTP] *****************************************************
changed: [machine1.xuwangwei.test]
changed: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Make sure One-Time Password is enabled if it's already defined] ****
skipping: [machine1.xuwangwei.test]
skipping: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Disable One-Time Password for on_master] ***************************
skipping: [machine1.xuwangwei.test]
skipping: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Test if IPA client has working krb5.keytab] ************************
ok: [machine1.xuwangwei.test]
ok: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Disable One-Time Password for client with working krb5.keytab] *****
skipping: [machine1.xuwangwei.test]
skipping: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Keytab or password is required for getting otp] ********************
skipping: [machine1.xuwangwei.test]
skipping: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Create temporary file for keytab] **********************************
skipping: [machine1.xuwangwei.test]
skipping: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Copy keytab to server temporary file] ******************************
skipping: [machine1.xuwangwei.test]
skipping: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Get One-Time Password for client enrollment] ***********************
skipping: [machine1.xuwangwei.test]
skipping: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Store the previously obtained OTP] *********************************
skipping: [machine1.xuwangwei.test]
skipping: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Remove keytab temporary file] **************************************
skipping: [machine1.xuwangwei.test]
skipping: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Store predefined OTP in admin_password] **************************************
skipping: [machine1.xuwangwei.test]
skipping: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Check if principal and keytab are set] *****************************
skipping: [machine1.xuwangwei.test]
skipping: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Check if one of password or keytabs are set] ***********************
skipping: [machine1.xuwangwei.test]
skipping: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - From host keytab, purge XUWANGWEI.TEST] ****************************
skipping: [machine1.xuwangwei.test]
skipping: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Backup and set hostname] *******************************************
changed: [machine2.xuwangwei.test]
changed: [machine1.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Create temporary krb5 configuration] *******************************
ok: [machine2.xuwangwei.test]
ok: [machine1.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Join IPA] **********************************************************
changed: [machine1.xuwangwei.test]
changed: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : The krb5 configuration is not correct] ***************************************
skipping: [machine1.xuwangwei.test]
skipping: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : IPA test failed] *************************************************************
skipping: [machine1.xuwangwei.test]
skipping: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Fail due to missing ca.crt file] *********************************************
skipping: [machine1.xuwangwei.test]
skipping: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Configure IPA default.conf] ****************************************
changed: [machine1.xuwangwei.test]
changed: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Configure SSSD] ****************************************************
changed: [machine1.xuwangwei.test]
changed: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - IPA API calls for remaining enrollment parts] **********************
changed: [machine1.xuwangwei.test]
changed: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Fix IPA ca] ********************************************************
skipping: [machine1.xuwangwei.test]
skipping: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Create IPA NSS database] *******************************************
changed: [machine1.xuwangwei.test]
changed: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Configure SSH and SSHD] ********************************************
changed: [machine1.xuwangwei.test]
changed: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Configure automount] ***********************************************
ok: [machine1.xuwangwei.test]
ok: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Configure firefox] *************************************************
skipping: [machine1.xuwangwei.test]
skipping: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Configure NIS] *****************************************************
changed: [machine2.xuwangwei.test]
changed: [machine1.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Remove temporary krb5.conf] **************************************************
changed: [machine1.xuwangwei.test]
changed: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Configure krb5 for IPA realm] **************************************
changed: [machine2.xuwangwei.test]
changed: [machine1.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Configure certmonger] **********************************************
changed: [machine1.xuwangwei.test]
changed: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Install - Restore original admin password if overwritten by OTP] *************
skipping: [machine1.xuwangwei.test]
skipping: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Cleanup leftover ccache] *****************************************************
ok: [machine2.xuwangwei.test]
ok: [machine1.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Remove temporary krb5.conf] **************************************************
ok: [machine1.xuwangwei.test]
ok: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Remove temporary krb5.conf backup] *******************************************
changed: [machine1.xuwangwei.test]
changed: [machine2.xuwangwei.test]

TASK [freeipa.ansible_freeipa.ipaclient : Uninstall IPA client] ********************************************************
skipping: [machine1.xuwangwei.test]
skipping: [machine2.xuwangwei.test]

PLAY RECAP *************************************************************************************************************
ipa-server.xuwangwei.test  : ok=53   changed=32   unreachable=0    failed=0    skipped=38   rescued=0    ignored=0
machine1.xuwangwei.test    : ok=28   changed=17   unreachable=0    failed=0    skipped=25   rescued=0    ignored=0
machine2.xuwangwei.test    : ok=28   changed=17   unreachable=0    failed=0    skipped=25   rescued=0    ignored=0

```

至此，服务端和客户端已安装完毕。

[^1]: Ansible 管理节点是指安装 Ansible 和运行 Ansible 控制命令的主机。
[^2]: 目标节点是指需要部署 FreeIPA 的主机，Ansible 会通过 SSH 对这些节点进行操作。