---
date: 
  created: 2024-06-13
  updated: 2025-03-24
categories: 
  - 好物分享
draft: false
---

# VMware Workstation 和 VMware Fusion 对所有用户完全免费

<!-- more -->

!!! question "从什么时候开始对个人用户完全免费"

    2024 年 5 月 13 号，VMware 官方发布了一篇[博客](https://blogs.vmware.com/teamfusion/2024/05/fusion-pro-now-available-free-for-personal-use.html)，宣布 VMware Workstation Pro 和 VMware Fusion Pro 对个人用户完全免费。

!!! question "从什么时候开始对所有用户完全免费"

    2024 年 11 月 11 号，VMware 官方发布了一篇[博客](https://blogs.vmware.com/cloud-foundation/2024/11/11/vmware-fusion-and-workstation-are-now-free-for-all-users/)，宣布 VMware Workstation Pro 和 VMware Fusion Pro 对所有用户完全免费。

具体到版本的话，VMware Workstation Pro 从 17.6.2 版本开始，VMware Fusion Pro 从 13.6.2 版本开始，不再需要许可证密钥，所有用户可以免费使用。

下面是截取自 VMware Workstation Pro 17.6.2 版本的[发行说明](https://techdocs.broadcom.com/cn/zh-cn/vmware-cis/desktop-hypervisors/workstation-pro/17-0/release-notes/vmware-workstation-1762-pro-release-notes.html)

!!! quote

    VMware Workstation Pro 不再需要许可证密钥，现在可免费用于商业、教育和个人用途。

下面是截取自 VMware Fusion Pro 13.6.2 版本的[发行说明](https://techdocs.broadcom.com/cn/zh-cn/vmware-cis/desktop-hypervisors/fusion-pro/13-0/release-notes/vmware-fusion-1362-release-notes.html)

!!! quote

    VMware Fusion 不再需要许可证密钥，现在可免费用于商业、教育和个人用途。

## 如何下载

### ~~方式一，直接下载~~

!!! failure "链接已失效"

    2025 年 3 月 27 日，我使用下面的链接下载时，会被重定向到博通的官网，已经无法直接下载了，太恶心了。之前我都是通过下面的链接下载的，因为这是自动更新时暴露出来的地址，现在博通这么搞，就是不想让我们直接下载了，必须注册账号才能下载了。

=== "Windows"

    ```text
    https://softwareupdate.vmware.com/cds/vmw-desktop/ws/17.6.2/24409262/windows/core/VMware-workstation-17.6.2-24409262.exe.tar
    ```

=== "Linux"

    ```text
    https://softwareupdate.vmware.com/cds/vmw-desktop/ws/17.6.2/24409262/linux/core/VMware-Workstation-17.6.2-24409262.x86_64.bundle.tar
    ```

=== "macOS"

    ```text
    https://softwareupdate.vmware.com/cds/vmw-desktop/fusion/13.6.2/24409261/universal/core/com.vmware.fusion.zip.tar
    ```

### 方式二，官网注册账号下载

因为 2023 年 11 月 22 号 Broadcom(博通)完成了对 VMware 的收购，所以现在要去[博通官网](https://www.broadcom.cn/)注册账号才能下载VMware Workstation Pro 和 VMware Fusion Pro。

1. 打开[博通官网](https://www.broadcom.cn/)后点击右上角的 Support Portal，在弹出的对话框中点击 Register 注册账号。如果你已经有了 VMware 账号，那么可以点击 Forgot Username/Password?，通过重置 VMware 账号的密码的方式，继续使用原来的 VMware 账号。

    ![图片1](/assets/images/free_vmware/1.png)

2. 注册账号登录后，按照下图所示点击。

    ![图片2.png](/assets/images/free_vmware/2.png)

3. 在搜索框中输入 fusion 或者 workstation 就能过滤出我们要下载的软件，如下图所示。

    !!! info

        VMware Fusion 是在 macOS 上运行的，VMware Workstation 是在 Windows 上运行的，不要搞错了。

    ![图片3.png](/assets/images/free_vmware/3.png)

    ![图片4.png](/assets/images/free_vmware/4.png)

    下面是 Workstation 的例子，和上面 Fusion 差不多。

    ![图片5.png](/assets/images/free_vmware/5.png)

    ![图片6.png](/assets/images/free_vmware/6.png)

## 已有 VMware 账号如何转移到 Broadcom(博通)

!!! quote

    方法来自官方 KB366920 文档 [VMware migrated users account activation issues](https://knowledge.broadcom.com/external/article?articleId=366920)

1. 打开博通官网后点击右上角的 Support Portal，在弹出的对话框中点击 Forgot Username/Password?。

    ![图片7.png](/assets/images/free_vmware/7.png)

2. 输入已有的 VMware 账号

    ![图片8.png](/assets/images/free_vmware/8.png)

    ![图片9.png](/assets/images/free_vmware/9.png)

3. 接着你的邮箱会收到重置密码的链接，点开链接输入新的密码即可实现 VMware 账号转移到 Broadcom(博通)。

