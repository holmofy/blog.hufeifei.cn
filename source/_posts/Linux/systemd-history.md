---
title: SysVinit -> Upstart -> Systemd
date: 2025-04-08
categories: Linux运维
tags: Linux
---

我记得很早之前，我写过一篇[文章](https://blog.hufeifei.cn/2017/04/Linux/CentOS-7-Systemd/)，介绍CentOS中引入的Systemd，后面我又[从CentOS切换到了Debian](https://blog.hufeifei.cn/2025/02/Linux/debian/)，今天服务器又碰到了一个事故，用ChatGPT重温了一下Systemd的演变历史。

Linux 启动系统（init system）的演进过程体现了 Linux 从“能用”到“高效现代”的重大变革。

| 发行版           | 初期使用     | 过渡期         | 现今使用                  |
| ------------- | -------- | ----------- | --------------------- |
| Debian        | SysVinit | Upstart（短暂） | systemd（从 Debian 8 起） |
| Ubuntu        | SysVinit | Upstart     | systemd（从 15.04 起）    |
| CentOS / RHEL | SysVinit | -           | systemd（从 7 起）        |
| Arch Linux    | SysVinit | -           | systemd（从 2012 起）     |

下面从历史背景、设计动因、关键技术变化、影响等多个角度，详细讲解：

---

## 🕰️ 一、SysVinit：经典但老旧（1980s - \~2010）

### 📌 背景

* 起源于 UNIX System V，Linux 借鉴使用。
* 是 Linux 在 1990s 到 2000s 期间的主流 init 系统。

### ⚙️ 特点

* 启动脚本：位于 `/etc/init.d/`，每个服务一个脚本。
* 控制启动顺序靠软链接：如 `/etc/rc5.d/S20nginx`
* 运行级别（runlevel）控制系统状态，例如：

  * 1：单用户模式
  * 3：文本多用户
  * 5：图形界面

### ⚠️ 问题

1. **串行启动**，速度慢。
2. **无依赖管理**，脚本靠人为安排执行顺序。
3. **服务无监控机制**，崩了不会自动拉起。
4. **脚本不统一**，难维护、难调试。

---

## 🔁 二、Upstart：向事件驱动演进（2006 - \~2014）

### 📌 背景

* Ubuntu 开发，为了解决 SysVinit 的性能与管理瓶颈。
* 最初用于 Ubuntu 6.10（2006），一直用到 Ubuntu 14.10。

### ⚙️ 特点

* **事件驱动模型**：比如“网络上线”、“磁盘挂载完成”作为服务启动的条件。
* **异步启动服务**，并行度更高。
* 向后兼容 `/etc/init.d/` 脚本。

### 📄 配置文件

配置文件位于 `/etc/init/`，如：

```bash
start on filesystem and net-device-up IFACE=eth0
stop on runlevel [!2345]
exec /usr/sbin/sshd
```

### 👍 优点

* 启动快；
* 更智能的服务启动条件；
* 初步引入服务状态查询（`start`/`stop`/`status`）。

### ⚠️ 缺点

1. **事件模型复杂**，调试困难；
2. **不统一**：日志、挂载等还是用其他工具；
3. **依赖链分析弱**；
4. **Red Hat 未采纳**，生态影响力有限。

---

## 🚀 三、systemd：现代初始化系统（2010 - 至今）

### 📌 背景

* 由 Red Hat 的 Lennart Poettering 和 Kay Sievers 开发。
* 旨在统一 Linux 的服务、挂载、日志、会话等系统组件。
* 2010 年首次发布，快速成为主流（Debian、Red Hat、Arch 全部迁移）。

### 🧠 设计理念

* 一切都是 unit（单元）：服务、挂载点、socket、设备、计时器……
* 统一管理、并发启动、状态追踪、自动重启、cgroup 限制、日志收集。

### ⚙️ 重大技术特点

| 特性           | 描述                                       |
| ------------ | ---------------------------------------- |
| 并行启动         | 利用服务依赖图（依赖解析）+ 并发执行，大幅提升启动速度             |
| 单元文件（Unit）   | `.service`、`.mount`、`.socket` 等格式统一的配置文件 |
| 状态管理         | 所有服务都有状态（active, failed, dead）可控         |
| `journalctl` | 内置日志系统，服务日志集中、结构化                        |
| Socket 激活    | 类似 xinetd，服务按需自动激活                       |
| D-Bus 集成     | 服务可以通过 D-Bus 动态激活                        |
| CGroup 管理    | 使用 Linux Control Groups 限制资源，系统级沙箱控制     |
| `timer` 单元   | 替代 cron，支持精准调度、依赖执行                      |

### 📄 配置文件例子（nginx.service）：

```ini
[Unit]
Description=NGINX web server
After=network.target

[Service]
ExecStart=/usr/sbin/nginx
ExecReload=/usr/sbin/nginx -s reload
ExecStop=/usr/sbin/nginx -s quit
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### 👍 优点

1. 启动快、依赖明确；
2. 管理统一、工具完善（systemctl、journalctl）；
3. 服务更稳定可靠（失败重启、健康检查）；
4. 支持现代 Linux 特性（容器、沙箱、用户服务等）；

### ⚠️ 批评与争议

* 设计复杂、模块庞大，有人觉得它\*\*“违背 Unix 哲学”\*\*；
* 破坏兼容性，老工具和脚本需要迁移；
* 太强势：替代了 cron、syslog、init 等多个工具。

---

## 📊 演进路径一览表

| 年代      | 系统       | 主导者           | 特点简述              |
| ------- | -------- | ------------- | ----------------- |
| \~1980s | SysVinit | UNIX System V | 脚本驱动、运行级别、串行启动    |
| 2006    | Upstart  | Ubuntu        | 事件驱动、异步启动、兼容 SysV |
| 2010 至今 | systemd  | Red Hat       | 并发启动、统一管理、现代特性丰富  |

---

## 🧩 总结对比

| 特性        | SysVinit  | Upstart   | systemd         |
| --------- | --------- | --------- | --------------- |
| 启动方式      | 串行        | 事件驱动 + 异步 | 依赖驱动 + 并发启动     |
| 配置方式      | Shell 脚本  | 自定义语法     | INI 风格 unit 文件  |
| 服务监控      | 无         | 有限        | 完善（自动重启、状态）     |
| 日志支持      | 外部 syslog | 外部 syslog | 内置 `journalctl` |
| 服务依赖管理    | 靠人为软链接顺序  | 初步支持      | 强依赖图支持          |
| socket 激活 | 无         | 支持        | 支持              |
| 生态兼容      | 高         | 一般        | 有破坏性，影响广泛       |

---

在 Linux 从 **SysVinit → Upstart → systemd** 的演进过程中，**运维命令和思维方式也发生了重大变革**。这些变革不仅仅是命令名称的变化，更是**系统控制理念、服务管理方式、日志处理方法和自动化能力的质变**。

## 🧱 一、SysVinit 时代（传统脚本式运维）

### 🧰 运维命令（分发式、手工化）：

```bash
# 启动服务
/etc/init.d/sshd start

# 停止服务
/etc/init.d/sshd stop

# 查看服务是否运行（通常要靠 ps）
ps aux | grep nginx

# 查看日志（只能看 log 文件）
tail -f /var/log/messages
tail -f /var/log/nginx/access.log

# 设置开机启动
chkconfig sshd on           # RedHat系
update-rc.d sshd defaults   # Debian系
```

### 🧩 特点：

* 命令风格分发式，每个命令管理一个功能（chkconfig、service、ps、tail）。
* 各种行为靠脚本或手动组合。
* 没有统一“服务状态”和“日志管理”的概念。

---

## 🔁 二、Upstart 时代（事件驱动，命令初步统一）

### 🧰 运维命令：

```bash
# 启动服务
start ssh

# 停止服务
stop ssh

# 重启服务
restart ssh

# 查看状态
status ssh

# 配置文件：/etc/init/ssh.conf
```

### 🧩 特点：

* 命令更统一，但仅限 Upstart 服务；
* 日志仍然靠传统 syslog；
* 服务依然不能“自愈”，不具备自动重启等能力；
* 某些服务仍得通过 `/etc/init.d/` 控制。

---

## 🚀 三、systemd 时代（集中管理、状态感知、自动化）

### 🧰 核心命令：`systemctl`（统一接口）

```bash
# 启动 / 停止 / 重启服务
systemctl start nginx
systemctl stop nginx
systemctl restart nginx

# 查看服务状态（非常详细）
systemctl status nginx

# 设置服务开机启动
systemctl enable nginx

# 禁用开机启动
systemctl disable nginx

# 重新加载服务配置
systemctl daemon-reexec
```

### 🧾 日志统一：`journalctl`

```bash
# 查看某服务日志
journalctl -u nginx

# 查看最近日志（自动分页）
journalctl -xe

# 实时查看日志（tail 模式）
journalctl -fu nginx

# 开机以来所有日志
journalctl -b
```

### ⚙️ 系统层级控制

```bash
# 显示当前系统运行状态
systemctl is-system-running

# 设置默认启动目标（相当于运行级别）
systemctl set-default multi-user.target
```

---

## 🔁 运维视角的重大变化总结

| 变化点       | SysVinit                | systemd                              | 改变价值       |
| --------- | ----------------------- | ------------------------------------ | ---------- |
| 启停命令      | 脚本调用                    | 统一 systemctl                         | 易于记忆，统一管理  |
| 服务状态      | 基本无                     | 多状态（active、failed等）                  | 更精细的运维反馈   |
| 开机启动管理    | chkconfig / update-rc.d | systemctl enable/disable             | 简洁、强大、统一   |
| 日志管理      | syslog、tail 文件          | journalctl                           | 结构化、查询更强   |
| 自动重启服务    | 无                       | Restart=on-failure 等选项               | 服务自愈能力     |
| 并发启动与依赖管理 | 无，靠脚本顺序                 | 内建依赖树与并发调度                           | 启动更快、可靠性更强 |
| 系统资源隔离    | 无                       | 使用 CGroup                            | 服务粒度资源管控   |
| 运维自动化     | 复杂脚本+crontab            | `systemd-timer` + `unit` + `oneshot` | 统一调度+逻辑更清晰 |

---

## 🛠️ 小例子：一行命令对比

| 场景          | SysVinit                           | systemd                  |                          |
| ----------- | ---------------------------------- | ------------------------ | ------------------------ |
| 启动 nginx    | `/etc/init.d/nginx start`          | `systemctl start nginx`  |                          |
| 查看 nginx 状态 | \`ps aux                           | grep nginx\`             | `systemctl status nginx` |
| 查看 nginx 日志 | `tail -f /var/log/nginx/error.log` | `journalctl -u nginx -f` |                          |
| 设置开机启动      | `chkconfig nginx on`               | `systemctl enable nginx` |                          |

---

## ✨ 总结一句话：

> 从 **“手动控制、分发命令”** 到 **“统一接口、状态感知、自动恢复”**，`systemd` 把 Linux 运维推入了一个可视化、自动化、结构化的现代时代。

以下是 **Linux 运维常用命令对照表**，重点对比了 **SysVinit/传统命令** 和 **systemd 时代**对应的命令和操作，方便你快速迁移和理解。

# 🚀 Linux 运维命令对照表（SysVinit/传统 vs systemd）

| 任务类别       | 传统 SysVinit / 其他命令                                            | systemd 时代命令 / 工具                               | 说明 / 备注                              |
| ---------- | ------------------------------------------------------------- | ----------------------------------------------- | ------------------------------------ |
| **服务管理**   | `/etc/init.d/nginx start`                                     | `systemctl start nginx`                         | 启动服务                                 |
|            | `/etc/init.d/nginx stop`                                      | `systemctl stop nginx`                          | 停止服务                                 |
|            | `/etc/init.d/nginx restart`                                   | `systemctl restart nginx`                       | 重启服务                                 |
|            | `/etc/init.d/nginx status`                                    | `systemctl status nginx`                        | 查看服务状态                               |
|            | `service nginx start`                                         | 也可用 `systemctl`                                 | 部分系统兼容旧命令                            |
|            | `chkconfig nginx on` / `update-rc.d nginx enable`             | `systemctl enable nginx`                        | 设置服务开机启动                             |
|            | `chkconfig nginx off` / `update-rc.d nginx disable`           | `systemctl disable nginx`                       | 取消开机启动                               |
| **服务日志**   | `tail -f /var/log/nginx/error.log`                            | `journalctl -u nginx -f`                        | 查看服务实时日志                             |
|            | `cat /var/log/messages`                                       | `journalctl`                                    | 查看系统日志                               |
| **系统启动管理** | 运行级别切换 `init 3` / `telinit 3`                                 | `systemctl isolate multi-user.target`           | 切换运行级别 / 目标状态                        |
|            | 查看运行级别 `runlevel`                                             | `systemctl get-default`                         | 查看默认启动目标                             |
|            |                                                               | `systemctl set-default graphical.target`        | 设置默认启动目标（多用户或图形）                     |
| **网络管理**   | `ifconfig` / `ip addr` / `route`                              | `ip addr` / `ip route` / `networkctl`           | `networkctl` 需启用 systemd-networkd 服务 |
|            | `/etc/network/interfaces` 或 `/etc/sysconfig/network-scripts/` | `/etc/systemd/network/*.network`                | 网络配置文件路径不同                           |
|            | `service network restart`                                     | `systemctl restart systemd-networkd`            | 重启网络服务                               |
|            | `nmcli`                                                       | `nmcli`（NetworkManager CLI，独立工具）                | 桌面系统多用 NetworkManager 管理网络           |
| **磁盘挂载**   | `mount / umount`                                              | `mount / umount` / `systemctl start foo.mount`  | 挂载命令一致；systemd 可管理挂载点为 unit          |
|            | 编辑 `/etc/fstab`                                               | 编辑 `/etc/fstab` 或 systemd mount unit 文件         | 挂载配置方式有扩展                            |
|            |                                                               | `systemctl daemon-reexec`                       | 重新加载 systemd 配置                      |
| **进程管理**   | `ps aux` / `kill PID`                                         | 依然使用 `ps`、`kill`                                | 进程命令无变化                              |
|            |                                                               | `systemctl kill service-name`                   | 杀死整个服务控制组内所有进程                       |
| **CPU 管理** | `nice` / `renice` / `taskset`                                 | `systemctl set-property service CPUQuota=50%`   | 支持服务级别 CPU 限制                        |
| **内存管理**   | `free` / `vmstat`                                             | `systemctl set-property service MemoryMax=512M` | 服务级内存限制                              |
| **定时任务**   | `crontab -e`                                                  | `systemd timers`                                | 支持依赖、日志及状态查看                         |
|            | `cron`                                                        | `systemctl list-timers`                         | 列出定时任务                               |
|            |                                                               | `journalctl -u name.timer`                      | 查看定时任务日志                             |
| **日志管理**   | `/var/log/syslog` / `/var/log/messages`                       | `journalctl`                                    | 集中、结构化日志                             |
|            | `tail` / `grep` 等                                             | `journalctl -xe` / `journalctl -f`              | 支持丰富的过滤和实时查看                         |
| **系统信息**   | `uptime` / `free -m` / `df -h`                                | 命令不变，systemd 不替代                                | 传统系统信息命令依旧使用                         |

---

# 📌 附加 Tips

| 事项              | 命令举例                                  |
| --------------- | ------------------------------------- |
| 查看所有运行的服务       | `systemctl list-units --type=service` |
| 查看失败的服务         | `systemctl --failed`                  |
| 重新加载 systemd 配置 | `systemctl daemon-reload`             |
| 重启系统            | `systemctl reboot`                    |
| 关闭系统            | `systemctl poweroff`                  |

---

# 总结

| 特点     | SysVinit / 传统命令 | systemd / 现代命令      |
| ------ | --------------- | ------------------- |
| 命令统一性  | 命令多样，分散         | 统一 `systemctl`，集中管理 |
| 服务管理   | 脚本方式            | 单元文件、依赖管理           |
| 日志管理   | 分散日志文件          | `journalctl` 统一日志系统 |
| 资源管理   | 无               | 基于 CGroup 的细粒度资源限制  |
| 启动效率   | 串行启动            | 并行启动+依赖图            |
| 自动恢复能力 | 无               | 自动重启、失败检测           |

