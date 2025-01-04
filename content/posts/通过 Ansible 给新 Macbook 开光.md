---
title: "通过 Ansible 给新 Macbook 开光"
date: 2025-01-04
categories: ['安装']
draft: false
---

> 说是开光，实际就是给新电脑安装各种命令行工具和软件，但又不想一个一个下载安装，所以就通过 Ansible 一键安装所有的软件。
>
> 开光一词是我从网上学来的，忘了是哪个人第一个提出。
>
> 通过 Ansible 给 Mac 安装软件是从《Ansible for DevOps》书里学来的。
>

## 开光
1. 安装 [Homebrew](https://brew.sh/)

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

2. 安装 [Ansible](https://ansible.readthedocs.io/projects/ansible-core/stable-2.18/)

```bash
brew install ansible
```

3. 创建工作目录 `~/dev/repo/setup-mac-by-ansible`

按你的命名目录的习惯创建工作目录即可，↑这是我的习惯。

```bash
mkdir -p ~/dev/repo/setup-mac-by-ansible && cd ~/dev/repo/setup-mac-by-ansible
```

4. 创建 playbook

playbook 是 Ansible 世界中的脚本，用于描述你希望设备最终达到的状态。我在这里使用 vim 创建并编辑名为 `setup-macos.yml` 的 playbook，你可以使用其他的文本编辑器。

```bash
vim setup-macos.yml
```

`setup-macos.yml` 文件内容如下：

```yaml
---
- hosts: localhost
  connection: local
  tasks:
    - name: Install Homebrew Formulae (CLI tools and utilities)
      community.general.homebrew:
        name:
          - hugo
          - mas
          - telnet
          - wget
          - zsh-autocomplete
          - zsh-syntax-highlighting
        state: present

    - name: Install Homebrew Casks (GUI apps)
      community.general.homebrew_cask:
        name:
          - adobe-acrobat-reader
          - aldente
          - appcleaner
          - beyond-compare
          - forklift
          - google-chrome
          - handbrake
          - imageoptim
          - jetbrains-toolbox
          - keka
          - mqttx
          - one-switch
          - popclip
          - shottr
          - tableplus
          - tailscale
          - vagrant
          - vagrant-vmware-utility
          - visual-studio-code
          - vmware-fusion
          - wireshark
        state: present

    - name: Install Mac App Store Applications
      community.general.mas:
        id:
          - 1355679052 # Dropover - Easier Drag & Drop
          - 1552536109 # PasteNow
          - 1443749478 # WPS
          - 441258766 # Magnet
          - 836500024 # WeChat
          - 1327661892 # Xmind
          - 1295203466 # Windows App
          - 425424353 # The Unarchiver
          - 524141863 # Jump Desktop
          - 1518036000 # Sequel Ace
        state: present

    - name: Configure zsh-syntax-highlighting in .zshrc
      ansible.builtin.blockinfile:
        path: ~/.zshrc
        create: true
        append_newline: true
        prepend_newline: true
        insertafter: EOF
        marker: "# {mark} ZSH-SYNTAX-HIGHLIGHTING"
        block: |
          if [ -f $HOMEBREW_PREFIX/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh ]; then
            source $HOMEBREW_PREFIX/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
          fi

    - name: Configure zsh-autocomplete in .zshrc
      ansible.builtin.blockinfile:
        path: ~/.zshrc
        create: true
        append_newline: true
        prepend_newline: true
        insertbefore: BOF
        marker: "# {mark} ZSH-AUTOCOMPLETE"
        block: |
          if [ -f $HOMEBREW_PREFIX/share/zsh-autocomplete/zsh-autocomplete.plugin.zsh ]; then
            source $HOMEBREW_PREFIX/share/zsh-autocomplete/zsh-autocomplete.plugin.zsh
          fi

    - name: Add gi() function to .zshrc for generating gitignore files
      ansible.builtin.lineinfile:
        path: ~/.zshrc
        create: true
        line: function gi() { curl -sLw "\n" https://www.toptal.com/developers/gitignore/api/$@ ;}

    - name: Generate global gitignore file using gi() function
      ansible.builtin.shell:
        cmd: source ~/.zshrc && gi macos,jetbrains,visualstudiocode >> ~/.gitignore_global && git config --global core.excludesfile ~/.gitignore_global
        creates: ~/.gitignore_global
        
    - name: Install Vagrant VMware Desktop plugin
      ansible.builtin.shell:
        cmd: vagrant plugin install vagrant-vmware-desktop

```

5. 一键安装

```bash
ansible-playbook setup-macos.yml -v
```

> 脚本写完后，最好先测试下，确定没问题后，再在新电脑上执行。
>
> 我是用虚拟机+老 Mac 测试的，因为虚拟机没法测试`Install Mac App Store Applications`这个 task，这个 task 是安装 Mac App Store 中的应用，必须要登录苹果账号，而虚拟机不支持，所以这个 task 单独在老 Mac 上跑的。如果你有更好的测试方法，别忘了告诉我。
>

---

## 使用 macOS 虚拟机测试
> Tart 是 macOS 上虚拟 macOS 的软件，且要求宿主机搭载 Apple M1/2/3/4 系列芯片。
>

1. 安装 [Tart](https://tart.run/)

```bash
brew install cirruslabs/cli/tart
```

2. 下载 [macOS 镜像](https://github.com/cirruslabs/macos-image-templates/pkgs/container/macos-sequoia-base)

```bash
tart clone ghcr.io/cirruslabs/macos-sequoia-base:latest sequoia-base
```

3. 启动 macOS 虚拟机

```bash
tart run sequoia-base
```

4. 给 macOS 虚拟机开光

按照上面写的开光步骤一步一步做就好了，有两点需要注意：一是 macOS 虚拟机中已经安装了 Homebrew，所以要跳过开光第一步；二是注释掉`Install Mac App Store Applications`这个 task。

脚本跑完后查看软件是不是都装好了，下面是我的脚本在虚拟机控制台的日志：

```plain
admin@admins-Virtual-Machine ~ % brew install ansible
==> Downloading https://ghcr.io/v2/homebrew/core/ansible/manifests/10.5.0-1
####################################################################################################################################################### 100.0%
==> Fetching dependencies for ansible: certifi, libsodium and libssh
==> Downloading https://ghcr.io/v2/homebrew/core/certifi/manifests/2024.8.30_1
####################################################################################################################################################### 100.0%
==> Fetching certifi
==> Downloading https://ghcr.io/v2/homebrew/core/certifi/blobs/sha256:1f1fc985a1c89bd40c73b17e3dfbf5483cb0417c8d4d12e2be66158a503ab169
####################################################################################################################################################### 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/libsodium/manifests/1.0.20
####################################################################################################################################################### 100.0%
==> Fetching libsodium
==> Downloading https://ghcr.io/v2/homebrew/core/libsodium/blobs/sha256:e8ba0aafe8fe7266d68630ff7ab11d7357af35dbf5113bb648a1e02bed397970
####################################################################################################################################################### 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/libssh/manifests/0.11.1
####################################################################################################################################################### 100.0%
==> Fetching libssh
==> Downloading https://ghcr.io/v2/homebrew/core/libssh/blobs/sha256:ef7a07e8b469c92123e7b81ee8356820e1e6866d8e05b278cdd544d1c7d4bf97
####################################################################################################################################################### 100.0%
==> Fetching ansible
==> Downloading https://ghcr.io/v2/homebrew/core/ansible/blobs/sha256:4135e4faf541afc2ae9886420fc109037701e1b961fb06495f642af7d3d9844a
####################################################################################################################################################### 100.0%
==> Installing dependencies for ansible: certifi, libsodium and libssh
==> Installing ansible dependency: certifi
==> Downloading https://ghcr.io/v2/homebrew/core/certifi/manifests/2024.8.30_1
Already downloaded: /Users/admin/Library/Caches/Homebrew/downloads/154ded4253bebd18cc03e7ed1d9a397ebad761afee0abdc83fc952c7586a49d6--certifi-2024.8.30_1.bottle_manifest.json
==> Pouring certifi--2024.8.30_1.arm64_sequoia.bottle.tar.gz
🍺  /opt/homebrew/Cellar/certifi/2024.8.30_1: 38 files, 34.6KB
==> Installing ansible dependency: libsodium
==> Downloading https://ghcr.io/v2/homebrew/core/libsodium/manifests/1.0.20
Already downloaded: /Users/admin/Library/Caches/Homebrew/downloads/a9a9a2e1207e214070682a14f6470fb686cbb6ba7c24c2c747c0ca0663f42557--libsodium-1.0.20.bottle_manifest.json
==> Pouring libsodium--1.0.20.arm64_sequoia.bottle.tar.gz
🍺  /opt/homebrew/Cellar/libsodium/1.0.20: 78 files, 883.4KB
==> Installing ansible dependency: libssh
==> Downloading https://ghcr.io/v2/homebrew/core/libssh/manifests/0.11.1
Already downloaded: /Users/admin/Library/Caches/Homebrew/downloads/3591f41514c0ebe698a36753dce13e71fa33363993429a04af0da249f1eea04c--libssh-0.11.1.bottle_manifest.json
==> Pouring libssh--0.11.1.arm64_sequoia.bottle.tar.gz
🍺  /opt/homebrew/Cellar/libssh/0.11.1: 25 files, 1.4MB
==> Installing ansible
==> Pouring ansible--10.5.0.arm64_sequoia.bottle.1.tar.gz
🍺  /opt/homebrew/Cellar/ansible/10.5.0: 32,696 files, 371.2MB
admin@admins-Virtual-Machine ~ % ansible-galaxy collection install community.general
Starting galaxy collection install process
Nothing to do. All requested collections are already installed. If you want to reinstall them, consider using `--force`.
admin@admins-Virtual-Machine ~ % mkdir -p ~/dev/repo/setup-mac-by-ansible && cd ~/dev/repo/setup-mac-by-ansible
admin@admins-Virtual-Machine setup-mac-by-ansible % vim setup-macos.yml
admin@admins-Virtual-Machine setup-mac-by-ansible % ansible-playbook setup-macos.yml -v
No config file found; using defaults
[WARNING]: No inventory was parsed, only implicit localhost is available
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [localhost] *********************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************
ok: [localhost]

TASK [Install Homebrew Formulae (CLI tools and utilities)] ***************************************************************************************************
changed: [localhost] => {"changed": true, "changed_pkgs": ["hugo", "mas", "telnet", "zsh-autocomplete", "zsh-syntax-highlighting"], "msg": "Changed: 5, Unchanged: 1", "unchanged_pkgs": ["wget"]}

TASK [Install Homebrew Casks (GUI apps)] *********************************************************************************************************************
changed: [localhost] => {"changed": true, "msg": "Changed: 21, Unchanged: 0"}

TASK [Configure zsh-syntax-highlighting in .zshrc] ***********************************************************************************************************
changed: [localhost] => {"changed": true, "msg": "File created"}

TASK [Configure zsh-autocomplete in .zshrc] ******************************************************************************************************************
changed: [localhost] => {"changed": true, "msg": "Block inserted"}

TASK [Add gi() function to .zshrc for generating gitignore files] ********************************************************************************************
changed: [localhost] => {"backup": "", "changed": true, "msg": "line added"}

TASK [Generate global gitignore file using gi() function] ****************************************************************************************************
changed: [localhost] => {"changed": true, "cmd": "source ~/.zshrc && gi macos,jetbrains,visualstudiocode >> ~/.gitignore_global && git config --global core.excludesfile ~/.gitignore_global", "delta": "0:00:00.530349", "end": "2025-01-03 18:49:07.134589", "msg": "", "rc": 0, "start": "2025-01-03 18:49:06.604240", "stderr": "/opt/homebrew/share/zsh-autocomplete/zsh-autocomplete.plugin.zsh: line 2: unsetopt: command not found\n/opt/homebrew/share/zsh-autocomplete/zsh-autocomplete.plugin.zsh: line 4: syntax error near unexpected token `)'\n/opt/homebrew/share/zsh-autocomplete/zsh-autocomplete.plugin.zsh: line 4: `() {'\n/opt/homebrew/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh: line 31: alias: -L: invalid option\nalias: usage: alias [-p] [name[=value] ... ]\n/opt/homebrew/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh: line 36: unalias: -m: invalid option\nunalias: usage: unalias [-a] name [name ...]\n/opt/homebrew/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh: line 39: 0=${(%):-%N}: bad substitution\n/opt/homebrew/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh: line 42: /.version: No such file or directory\n/opt/homebrew/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh: line 42: typeset: -g: invalid option\ntypeset: usage: typeset [-afFirtx] [-p] name[=value] ...\n/opt/homebrew/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh: line 43: /.revision-hash: No such file or directory\n/opt/homebrew/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh: line 43: typeset: -g: invalid option\ntypeset: usage: typeset [-afFirtx] [-p] name[=value] ...\n/opt/homebrew/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh: line 104: autoload: command not found\n/opt/homebrew/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh: line 105: is-at-least: command not found\n/opt/homebrew/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh: line 108: typeset: -g: invalid option\ntypeset: usage: typeset [-afFirtx] [-p] name[=value] ...\n/opt/homebrew/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh: line 112: typeset: -g: invalid option\ntypeset: usage: typeset [-afFirtx] [-p] name[=value] ...\n/opt/homebrew/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh: line 144: syntax error near unexpected token `;'\n/opt/homebrew/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh: line 144: `      (\" 0 0 fg=red, memo=zsh-syntax-highlighting\") ;&'", "stderr_lines": ["/opt/homebrew/share/zsh-autocomplete/zsh-autocomplete.plugin.zsh: line 2: unsetopt: command not found", "/opt/homebrew/share/zsh-autocomplete/zsh-autocomplete.plugin.zsh: line 4: syntax error near unexpected token `)'", "/opt/homebrew/share/zsh-autocomplete/zsh-autocomplete.plugin.zsh: line 4: `() {'", "/opt/homebrew/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh: line 31: alias: -L: invalid option", "alias: usage: alias [-p] [name[=value] ... ]", "/opt/homebrew/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh: line 36: unalias: -m: invalid option", "unalias: usage: unalias [-a] name [name ...]", "/opt/homebrew/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh: line 39: 0=${(%):-%N}: bad substitution", "/opt/homebrew/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh: line 42: /.version: No such file or directory", "/opt/homebrew/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh: line 42: typeset: -g: invalid option", "typeset: usage: typeset [-afFirtx] [-p] name[=value] ...", "/opt/homebrew/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh: line 43: /.revision-hash: No such file or directory", "/opt/homebrew/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh: line 43: typeset: -g: invalid option", "typeset: usage: typeset [-afFirtx] [-p] name[=value] ...", "/opt/homebrew/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh: line 104: autoload: command not found", "/opt/homebrew/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh: line 105: is-at-least: command not found", "/opt/homebrew/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh: line 108: typeset: -g: invalid option", "typeset: usage: typeset [-afFirtx] [-p] name[=value] ...", "/opt/homebrew/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh: line 112: typeset: -g: invalid option", "typeset: usage: typeset [-afFirtx] [-p] name[=value] ...", "/opt/homebrew/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh: line 144: syntax error near unexpected token `;'", "/opt/homebrew/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh: line 144: `      (\" 0 0 fg=red, memo=zsh-syntax-highlighting\") ;&'"], "stdout": "", "stdout_lines": []}

TASK [Install Vagrant VMware Desktop plugin] *****************************************************************************************************************
changed: [localhost] => {"changed": true, "cmd": "vagrant plugin install vagrant-vmware-desktop", "delta": "0:00:29.408146", "end": "2025-01-03 18:49:36.746833", "msg": "", "rc": 0, "start": "2025-01-03 18:49:07.338687", "stderr": "==> vagrant: A new version of Vagrant is available: 2.4.3 (installed version: 2.4.1)!\n==> vagrant: To upgrade visit: https://www.vagrantup.com/downloads.html", "stderr_lines": ["==> vagrant: A new version of Vagrant is available: 2.4.3 (installed version: 2.4.1)!", "==> vagrant: To upgrade visit: https://www.vagrantup.com/downloads.html"], "stdout": "Installing the 'vagrant-vmware-desktop' plugin. This can take a few minutes...\nInstalled the plugin 'vagrant-vmware-desktop (3.0.4)'!", "stdout_lines": ["Installing the 'vagrant-vmware-desktop' plugin. This can take a few minutes...", "Installed the plugin 'vagrant-vmware-desktop (3.0.4)'!"]}

PLAY RECAP ***************************************************************************************************************************************************
localhost                  : ok=8    changed=7    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   


```

---

## 补充
### 宿主机 ssh 登录虚拟机
虚拟机的账号和密码都是 admin

```bash
ssh admin@$(tart ip sequoia-base)
```

### 重置虚拟机
```bash
tart delete sequoia-base
tart clone ghcr.io/cirruslabs/macos-sequoia-base:latest sequoia-base
```

### 我没有新 Mac
没买呢，提前准备好脚本，未来哪天买了，就能直接用了。

