---
title: "VMware Workstation 和 VMware Fusion 对所有用户完全免费"
date: 2024-06-13
categories: ['好物分享']
draft: false
description: VMware Workstation Pro和VMware Fusion Pro从2024年5月13日起对个人用户免费，2024年11月11日起对所有用户免费，包括商业、教育和个人用途。从Workstation Pro 17.6.2和Fusion Pro 13.6.2版本开始，用户无需许可证密钥即可免费使用。用户可通过浏览器或wget直接下载，或通过博通官网注册账号获取。由于博通收购了VMware，现有VMware账号需通过密码重置迁移至博通账号。
---


## 从什么时候开始对个人用户完全免费
2024 年 5 月 13 号，VMware 官方发布了一篇[博客](https://blogs.vmware.com/teamfusion/2024/05/fusion-pro-now-available-free-for-personal-use.html)，宣布 VMware Workstation Pro 和 VMware Fusion Pro 对个人用户完全免费。
## 从什么时候开始对所有用户完全免费
2024 年 11 月 11 号，VMware 官方发布了一篇[博客](https://blogs.vmware.com/cloud-foundation/2024/11/11/vmware-fusion-and-workstation-are-now-free-for-all-users/)，宣布 VMware Workstation Pro 和 VMware Fusion Pro 对所有用户完全免费。

具体到版本的话，VMware Workstation Pro 从 17.6.2 版本开始，VMware Fusion Pro 从 13.6.2 版本开始，不再需要许可证密钥，所有用户可以免费使用。

下面是截取自 VMware Workstation Pro 17.6.2 版本的[发行说明](https://techdocs.broadcom.com/us/en/vmware-cis/desktop-hypervisors/workstation-pro/17-0/release-notes/vmware-workstation-1762-pro-release-notes.html)
> VMware Workstation Pro no longer requires a license key and is now free for commercial, educational, and personal use.

下面是截取自 VMware Fusion Pro 13.6.2 版本的[发行说明](https://techdocs.broadcom.com/us/en/vmware-cis/desktop-hypervisors/fusion-pro/13-0/release-notes/vmware-fusion-1362-release-notes.html)
> VMware Fusion no longer requires a license key and is now free for commercial, educational, and personal use.
## 如何下载
### 方式一，直接使用浏览器或 wget 下载
#### VMware Workstation Pro 17.6.2 下载地址
##### Windows
```text
https://softwareupdate.vmware.com/cds/vmw-desktop/ws/17.6.2/24409262/windows/core/VMware-workstation-17.6.2-24409262.exe.tar
```
##### Linux
```text
https://softwareupdate.vmware.com/cds/vmw-desktop/ws/17.6.2/24409262/linux/core/VMware-Workstation-17.6.2-24409262.x86_64.bundle.tar
```
#### VMware Fusion Pro 13.6.2 下载地址
##### macOS
```text
https://softwareupdate.vmware.com/cds/vmw-desktop/fusion/13.6.2/24409261/universal/core/com.vmware.fusion.zip.tar
```

### ~~方式二，官网注册账号下载~~
因为 2023 年 11 月 22 号 Broadcom(博通)完成了对 VMware 的收购，所以现在要去[博通官网](https://www.broadcom.cn/)注册账号才能下载VMware Workstation Pro 和 VMware Fusion Pro。

1. 打开[博通官网](https://www.broadcom.cn/)后点击右上角的 Support Portal，在弹出的对话框中点击 Register 注册账号。如果你已经有了 VMware 账号，那么可以点击 Forgot Username/Password?，通过重置 VMware 账号的密码的方式，继续使用原来的 VMware 账号。

![图片1](/images/free_vmware/1.png)

2. 注册账号登录后，按照下图所示点击。

![图片2.png](/images/free_vmware/2.png)

3. 在搜索框中输入 fusion 或者 workstation 就能过滤出我们要下载的软件，如下图所示。
> VMware Fusion 是在 macOS 上运行的，VMware Workstation 是在 Windows 上运行的，不要搞错了。

![图片3.png](/images/free_vmware/3.png)
![图片4.png](/images/free_vmware/4.png)
下面是 Workstation 的例子，和上面 Fusion 差不多。
![图片5.png](/images/free_vmware/5.png)
![图片6.png](/images/free_vmware/6.png)
## ~~已有 VMware 账号如何转移到 Broadcom(博通)~~
> 方法来自官方 KB366920 文档[VMware migrated users account activation issues](https://knowledge.broadcom.com/external/article?articleId=366920)

1. 打开博通官网后点击右上角的 Support Portal，在弹出的对话框中点击 Forgot Username/Password?。

![图片7.png](/images/free_vmware/7.png)

2. 输入已有的 VMware 账号

![图片8.png](/images/free_vmware/8.png)
![图片9.png](/images/free_vmware/9.png)

3. 接着你的邮箱会收到重置密码的链接，点开链接输入新的密码即可实现 VMware 账号转移到 Broadcom(博通)。

