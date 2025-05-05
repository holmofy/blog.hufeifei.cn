---
title: 用debian替代CentOS
date: 2025-02-13
categories: Linux运维
tags: Linux
---

2021年写过[一篇文章](https://blog.hufeifei.cn/2021/02/Linux/centos7/)解决自己服务器CentOS7 yum源中软件版本过低的问题。到2025年，选择 Debian 还是 CentOS 已经不需要再多说了。

2020年，红帽公司宣布，将在2021年12月31日和2024年6月30日分别[终止对CentOS 8和CentOS 7的服务支持](https://www.centos.org/centos-linux-eol/)，把CentOS项目的工作和投资集中在CentOS Stream上，以进一步推动Linux创新。

| 版本       | 发布日期       | 进入EOL停服阶段  |
|----------|------------|------------|
| CentOS 8 | 2019-09-24 | 2021-12-31 |
| CentOS 7 | 2014-07-07 | 2024-06-30 |
| CentOS 6 | 2011-11-27 | 2020-11-30 |
| CentOS 5 | 2007-04-12 | 2017-03-31 |

CentOS Stream 不再是 RHEL 的完全兼容版，而是介于 Fedora 和 RHEL 之间的“预览版”。更新比 RHEL 更快，但稳定性和保守程度比原 CentOS 差。变得更像测试渠道，不建议直接用于生产环境。

如果你是原本 CentOS 的目标用户，现在可以考虑：

|替代发行版|	简介|
|--------|------|
|AlmaLinux|	社区维护，100% RHEL 兼容，稳定、安全。|
|Rocky Linux|	CentOS 联合创始人打造，目标是“继任 CentOS”，生产级稳定。|
|Oracle Linux|	免费使用，强兼容 RHEL，但由 Oracle 主导，需权衡其商业背景。|

但是RHEL源码访问政策收紧，也影响了 Rocky Linux / AlmaLinux 这些替代品的信任度。

目前社区大部分都选择用Debian替代CentOS，这也是我考虑的方向。

## Debian

Debian 是一个由社区驱动的、完全开源的 GNU/Linux 操作系统，以其稳定、安全、自由著称。它是最早的 Linux 发行版之一，很多现代发行版都以 Debian 为基础。

* 诞生时间：1993 年
* 发起人：[Ian Murdock](https://en.wikipedia.org/wiki/Ian_Murdock)（“Debian” = Debra + Ian）
* 组织形式：非盈利组织 Debian Project
* 包管理系统：.deb 包 + APT（高级包工具）

Debian 相比CentOS有以下特点：
1. 稳定可靠: Debian Stable 分支 是出了名的稳定，广泛用于生产服务器。每个版本发布前都经历长时间测试，更新保守但安全。
2. 完全开源和自由: 所有官方软件包都必须遵守 DFSG (Debian 自由软件指南)。非自由软件（如固件、驱动）被严格区分为 contrib / non-free 仓库。
3. 包管理强大: 使用 .deb + dpkg + apt，可以便捷地安装、升级、管理成千上万的软件包。支持依赖自动解决、配置文件管理、自动更新。
4. 跨架构支持极广: 支持超过 10 个架构：amd64、arm64、armhf、i386、riscv64 等。广泛用于 PC、树莓派、嵌入式设备甚至太空站。
5. 社区驱动，不依赖商业公司: Debian 项目由志愿者维护，不属于任何公司，不受商业利益驱动。决策由民主投票机制决定，开发者需通过身份验证。

Debian 有一套非常系统的版本发布节奏，按代号命名，每个版本都经历：

* 一个 **开发周期（unstable ➝ testing ➝ stable）**
* 一个 **稳定发布（stable）**
* 一个 **支持周期（约 5 年，含 LTS）**

下面是 Debian 近 10 个版本的详细列表，包括发布时间、代号、生命周期：

| 版本号  | 代号（Codename）   | 稳定版发布时间    | LTS 结束时间    | 备注                        |
| ---- | -------------- | ---------- | ----------- | ------------------------- |
| 2.2  | Potato         | 2000-08-15 | 停止支持        |                           |
| 3.0  | Woody          | 2002-07-19 | 停止支持        |                           |
| 3.1  | Sarge          | 2005-06-06 | 停止支持        |                           |
| 4.0  | Etch           | 2007-04-08 | 停止支持        |                           |
| 5.0  | Lenny          | 2009-02-14 | 2012-02（结束） | 第一次有 LTS（非官方）支持           |
| 6.0  | Squeeze        | 2011-02-06 | 2016-02-29  | 官方 LTS 起点                 |
| 7.0  | Wheezy         | 2013-05-04 | 2018-05-31  | systemd 可选                |
| 8.0  | Jessie         | 2015-04-26 | 2020-06-30  | 默认启用 systemd              |
| 9.0  | Stretch        | 2017-06-17 | 2022-06-30  |                           |
| 10.0 | Buster         | 2019-07-06 | 2024-06-30  | 当前仍处于 LTS 支持中             |
| 11.0 | Bullseye       | 2021-08-14 | 2026-06（预计） | 当前 LTS 用户推荐               |
| 12.0 | Bookworm       | 2023-06-10 | 2028-06（预计） | 当前最新稳定版                   |
| 13.0 | Trixie *(测试中)* | 预计 2025    | 预计 2030     | 处于 testing 分支             |
| 14.0 | Forky *(未来)*   | -          | -           | sid (unstable) 的代号永远是 sid |

> * Debian 的代号都来自电影《Toy Story》里的角色。
> * `sid` 是一个角色名字（“破坏者小孩”），所以 Debian 的 **unstable 分支永远叫 sid**。
> * 每次正式发布的 stable 版本会从 sid/testing 分支中选出一个代号，并冻结、测试、发布。

## Debian扩展软件仓库

[之前那篇文章](https://blog.hufeifei.cn/2021/02/Linux/centos7/)中介绍了CentOS的扩展仓库，现在[官网都把这个页面归档了](https://wiki.centos.org/AdditionalResources(2f)Repositories.html)。

同样的，我找了一下Debian有没有扩展软件仓库：

| 目标   | CentOS 用法  | Debian 对应用法                        |
| ---- | ---------- | ---------------------------------- |
| 扩展包  | EPEL       | [Backports](https://wiki.debian.org/zh_CN/Backports) / [Sury](https://deb.sury.org/)(PHP相关的软件包)    |
| 多媒体  | Nux-Dextop | [Debian Multimedia](https://deb-multimedia.org/)    |
| 新版软件 | IUS        | [官方 backports](https://backports.debian.org/) / [Ubuntu PPA](https://launchpad.net/ubuntu/+ppas)   |

相比之下 Debian 的第三方仓库生态不如 CentOS（或说 RHEL 系列）那样成熟、集中、企业级化，没有像CentOS的[EPEL](https://docs.fedoraproject.org/en-US/epel/)、[Remi](https://rpms.remirepo.net/)、[IUS](https://ius.io/)、[ELRepo](https://elrepo.org/wiki/doku.php?id=start)这样被广泛使用，社区文档多的第三方仓库生态。像 IUS、ELRepo 背后有企业或成熟团队支持。

Debian的第三方仓库比较分散，多为小型或个人维护项目，单一项目用途为主，比如[nginx](https://nginx.org/en/linux_packages.html#Debian)、[NodeJS](https://deb.nodesource.com/)、[Docker](https://docs.docker.com/engine/install/debian/)、[PostgreSQL](https://wiki.postgresql.org/wiki/Apt)等。

主要还是官方的为主，[这里](https://debian.pkgs.org/)有一个debian软件包查询网站，查询比较方便。
