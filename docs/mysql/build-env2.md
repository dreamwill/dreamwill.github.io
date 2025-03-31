---
date: 
  created: 2025-03-28
  updated: 2025-03-29
draft: false
---

# 搭建 MySQL 源码阅读和调试环境

!!! abstract

    教你 10 分钟搭建一个可以断点调试和阅读 MySQL 8.0 源代码的环境。
    
    你需要有一个运行 Debian 12 操作系统的虚拟机（下文中我都称它为远程主机），和一台安装了 Visual Studio Code 的个人电脑。我们会把 MySQL 程序和源代码安装在远程主机上，个人电脑只安装 VS Code 插件，远程断点调试。

!!! question "为什么使用 Ansible 搭建"

    因为 Ansible 可以保证幂等性，即无论执行多少次 Ansible 脚本，服务器的最终状态是一致的。Ansible 脚本采用 YAML 语法，描述我们希望服务器达到的状态。即使你不懂 Ansible 脚本也没关系，因为我们的目标是搭建环境，不是弄懂 Ansible 脚本，所以直接复制使用即可。

## 准备工作

安装 Ansible

```console
root@debian:~# apt install ansible
```

创建 Ansible 脚本

```console
root@debian:~# vi build_mysql.yml
```

将下面的脚本内容直接复制粘贴

```yaml title="build_mysql.yml"
---
- name: Build MySQL debug environment
  hosts: localhost
  become: true
  vars:
    mysql_root_password: MyNewPass4!
  tasks:
    - name: Install mysql-apt-config dependencies
      ansible.builtin.apt:
        name:
          - lsb-release
          - wget
          - gnupg
          - debconf-utils
        update_cache: true

    - name: Configure mysql-apt-config
      ansible.builtin.debconf:
        name: mysql-apt-config
        question: "{{ item.question }}"
        value: "{{ item.value }}"
        vtype: "{{ item.vtype }}"
      loop:
        - question: mysql-apt-config/select-connectors
          value: Disabled
          vtype: select
        - question: mysql-apt-config/select-server
          value: mysql-8.0
          vtype: select
        - question: mysql-apt-config/select-product
          value: Ok
          vtype: select

    - name: Install MySQL official repo
      ansible.builtin.apt:
        deb: https://repo.mysql.com/mysql-apt-config_0.8.33-1_all.deb

    - name: Configure mysql-community-server
      ansible.builtin.debconf:
        name: mysql-community-server
        question: "{{ item.question }}"
        value: "{{ item.value }}"
        vtype: "{{ item.vtype }}"
      loop:
        - question: mysql-community-server/root-pass
          value: "{{ mysql_root_password }}"
          vtype: password
        - question: mysql-community-server/re-root-pass
          value: "{{ mysql_root_password }}"
          vtype: password
        - question: mysql-server/default-auth-override
          value: Use Strong Password Encryption (RECOMMENDED)
          vtype: select

    - name: Install dependencies
      ansible.builtin.apt:
        name:
          - g++
          - gdb
          - tar
          - python3-pymysql
          - mysql-community-server-debug
          - mysql-community-server-debug-dbgsym
        update_cache: true
      
    - name: Edit my.cnf
      community.general.ini_file:
        path: /etc/mysql/mysql.conf.d/mysqld.cnf
        owner: root
        group: root
        mode: "0644"
        section: mysqld
        option: log_timestamps
        value: SYSTEM

    - name: Start service temporarily
      ansible.builtin.systemd_service:
        name: mysql
        state: started
        enabled: false
        daemon_reload: true

    - name: Create user 'example'@'%'
      community.mysql.mysql_user:
        login_user: root
        login_password: "{{ mysql_root_password }}"
        name: example
        plugin: caching_sha2_password
        plugin_auth_string: "{{ mysql_root_password }}"
        host: "%"
        priv: "*.*:ALL"

    - name: Stop service
      ansible.builtin.systemd_service:
        name: mysql
        state: stopped
        enabled: false

    - name: Get MySQL version
      ansible.builtin.shell: apt-cache show mysql-community-server-debug | grep '^Version:' | awk '{print $2}' | cut -d'-' -f1
      register: mysql_version
      changed_when: false
    
    - name: Get source
      ansible.builtin.shell: 
        cmd: wget -P /usr/src https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-{{ mysql_version.stdout }}.tar.gz
        creates: /usr/src/mysql-{{ mysql_version.stdout }}.tar.gz

    - name: Decompress source
      ansible.builtin.unarchive:
        src: /usr/src/mysql-{{ mysql_version.stdout }}.tar.gz
        dest: /usr/src/
        copy: false
        decrypt: false
```

执行脚本，静静等待完成即可，网速快的话大约 3 分钟就好了。

```console
root@debian:~# ansible-playbook build_mysql.yml 
```

??? success "控制台输出"

    ```text
    [WARNING]: No inventory was parsed, only implicit localhost is available
    [WARNING]: provided hosts list is empty, only localhost is available. Note that
    the implicit localhost does not match 'all'

    PLAY [Build MySQL debug environment] *********************************************

    TASK [Gathering Facts] ***********************************************************
    ok: [localhost]

    TASK [Install mysql-apt-config dependencies] *************************************
    changed: [localhost]

    TASK [Configure mysql-apt-config] ************************************************
    changed: [localhost] => (item={'question': 'mysql-apt-config/select-connectors', 'value': 'Disabled', 'vtype': 'select'})
    changed: [localhost] => (item={'question': 'mysql-apt-config/select-server', 'value': 'mysql-8.0', 'vtype': 'select'})
    changed: [localhost] => (item={'question': 'mysql-apt-config/select-product', 'value': 'Ok', 'vtype': 'select'})

    TASK [Install MySQL official repo] ***********************************************
    changed: [localhost]

    TASK [Configure mysql-community-server] ******************************************
    changed: [localhost] => (item={'question': 'mysql-community-server/root-pass', 'value': 'MyNewPass4!', 'vtype': 'password'})
    changed: [localhost] => (item={'question': 'mysql-community-server/re-root-pass', 'value': 'MyNewPass4!', 'vtype': 'password'})
    changed: [localhost] => (item={'question': 'mysql-server/default-auth-override', 'value': 'Use Strong Password Encryption (RECOMMENDED)', 'vtype': 'select'})

    TASK [Install dependencies] ******************************************************
    changed: [localhost]

    TASK [Edit my.cnf] ***************************************************************
    changed: [localhost]

    TASK [Start service temporarily] *************************************************
    changed: [localhost]

    TASK [Create user 'example'@'%'] *************************************************
    changed: [localhost]

    TASK [Stop service] **************************************************************
    changed: [localhost]

    TASK [Get MySQL version] *********************************************************
    ok: [localhost]

    TASK [Get source] ****************************************************************
    changed: [localhost]

    TASK [Decompress source] *********************************************************
    changed: [localhost]

    PLAY RECAP ***********************************************************************
    localhost                  : ok=13   changed=11   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
    ```

## 配置 Visual Studio Code

### 安装 Remote Development 插件

![Remote Development 插件截图](/assets/images/build_mysql/1.png)

### 远程登录

<video controls autoplay loop muted playsinline style="max-width: 100%;">
  <source src="/assets/anima/mysql/debian-remote-login.mp4" type="video/mp4">
  您的浏览器不支持视频标签。
</video>

我们使用上一步安装好的 Remote Development 插件打开远程主机的源代码目录，作为 VS Code 的工作区。MySQL 源代码在我们执行 Ansible 脚本的时候安装好了，不用担心。

### 安装 C/C++ Extension Pack 插件

![C/C++ Extension Pack 插件截图](/assets/images/build_mysql/4.png)

完成上一步的操作后，我们就把远程主机的源代码目录作为了 VS Code 的工作区。现在我们要在远程主机上安装 C/C++ Extension Pack 插件。在插件市场搜索到该插件后，会显示上图所示的安装在远程主机的按钮。

### 配置文件

最后，在工作区根目录下新建 .vscode 目录，然后在目录下新建 2 个 json 配置文件，文件内容按照我写的复制粘贴即可。完成这一步后，立刻就能调试了。

```json title=".vscode/launch.json"
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug MySQL",
            "type": "cppdbg",
            "request": "launch",
            "program": "/usr/sbin/mysqld-debug",
            "args": [
                "--user=mysql"
            ],
            "stopAtEntry": true,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "setupCommands": [
                {
                    "description": "Enable pretty-printing",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ]
        }
    ]
}
```

```json title=".vscode/settings.json"
{
    "C_Cpp.default.compilerPath": "/usr/bin/c++",
    "C_Cpp.errorSquiggles": "disabled"
}
```

![目录结构](/assets/images/build_mysql/5.png)
/// caption
目录结构
///

## 启动调试

<video controls autoplay loop muted playsinline style="max-width: 100%;">
  <source src="/assets/anima/mysql/debian-start-debug.mp4" type="video/mp4">
  您的浏览器不支持视频标签。
</video>

看，是不是非常简单:smile:。我们启动调试后，程序会在入口函数 main 处停下，等待我们的指令。你可以单击 ++f7++ 进入 main 函数，然后使用 ++f8++ 一步一步往下走。如果你不希望每次启动调试时，程序都停在 main 函数处，那么可以修改 `.vscode/launch.json` 中的 stopAtEntry 属性为 false。

下一篇，我们开始正式阅读 MySQL 源码。