---
title: GFW与翻墙
date: 2022-03-30
tags:
categories: 网络
---

[GFW](https://zh.wikipedia.org/wiki/防火长城)（Great Firewall of China，简称中国防火长城）是中国政府为实现网络审查而部署的一套强大而复杂的系统，旨在控制国内网络环境，屏蔽国外的互联网内容，保护国家的信息安全。GFW的技术演变与中国互联网治理政策的变化密切相关。从最初的简单封锁到如今深度包检测（DPI）和各种网络封锁手段的结合，GFW的技术也经历了多次更新和迭代。

翻墙软件的发展和协议的多样化，实际上与中国的网络封锁（也被称为“防火长城”）以及互联网审查的不断演进密切相关。从最初的简单绕过封锁方法到如今的多个加密协议，每个阶段的技术和协议都在一定程度上应对了不同的审查手段。

### 1. 最早的翻墙方法（2000年代初期）

GFW的雏形最早可以追溯到1990年代中期。随着互联网的普及，尤其是对外开放后的1995年，互联网在中国开始快速发展。然而，互联网带来了许多潜在的政治风险和信息安全问题，这促使中国政府开始进行网络管控。

* 1995年，中国信息安全技术研究中心开始研究如何通过技术手段对网络进行管控。
* 1997年，“[金盾工程](https://baike.baidu.com/item/金盾工程)”（Golden Shield Project）启动，该项目旨在建立一套包括网络监控、信息审查、网络封锁和控制在内的系统。这为后来的GFW提供了基础。

进入21世纪后，互联网的普及使得国外的网站和服务成为中国用户获取信息的主要途径。为了防止不符合政府审查标准的信息流入国内，中国政府开始对外部网站实施封锁。

* 2002年，中国政府开始屏蔽如BBC、Google等网站，以减少外国媒体的影响。
* 2003年，中国全面启动了**“防火长城”**（GFW）系统，主要通过域名解析和IP封锁来屏蔽敏感内容。此时，封锁主要集中在新闻和政治敏感网站，如西藏、法轮功、台湾等相关内容。

最早期的翻墙手段主要依赖于一些简单的代理服务器，如 [**SOCKS**](https://zh.wikipedia.org/wiki/SOCKS) 和 **HTTP(S) 代理**。这些协议比较简单，主要通过设置代理服务器来隐藏真实IP地址，绕过GFW（防火长城）的一些简单封锁。

- **SOCKS (4/4a/5)**：SOCKS 是一种应用层协议，广泛用于代理转发。在2000年代，SOCKS代理（特别是SOCKS5）常用于绕过封锁，但随着GFW的强化，它也逐渐被发现并封锁。
- **HTTP(S) 代理**：HTTP代理通过转发HTTP请求来实现翻墙，HTTPS则通过加密的形式提供安全的传输。不过，随着GFW的[深度包检测（DPI）](https://zh.wikipedia.org/wiki/深度包检测)技术的进步，HTTP代理逐渐无法有效突破封锁。

### 2. Shadowsocks（2012年左右）

[Shadowsocks](https://zh.wikipedia.org/wiki/Shadowsocks) 是由中国程序员**Clowwindy**（赵炯）在2012年开发的，代码开源在[Github](https://github.com/shadowsocks)，最初是作为一种简单的代理工具，用于绕过中国的网络审查。它的核心特性是 **Socks5代理**，结合了加密技术，使得流量不容易被GFW识别和封锁。

![shadowsocks](https://www.wizcase.com/wp-content/uploads/2018/12/Shadowsocks-500x150.png)

- **Shadowsocks** 采用了 **symmetric encryption**（对称加密）技术，例如AES加密，使得数据流在通过代理时得到了较好的隐蔽性，GFW也逐渐开始较难识别Shadowsocks流量。
- 随着使用的人数增多，Shadowsocks很快成为最主流的翻墙工具之一，并且衍生出了多种不同的实现（如ShadowsocksR，SSRR等）。

|Shadowsocks及其变体|状态|
|---|---|
|![shadowsocks](https://img.shields.io/github/stars/shadowsocks/shadowsocks)[shadowsocks](https://github.com/shadowsocks/shadowsocks) | 原始Shadowsocks项目，由Python编写，项目已被清空 |
|![shadowsocks](https://img.shields.io/github/stars/shadowsocks/shadowsocks-libev)[shadowsocks-libev](https://github.com/shadowsocks/shadowsocks-libev) | C语言编写，目前仅修复BUG |
|![shadowsocks](https://img.shields.io/github/stars/shadowsocks/shadowsocks-rust)[shadowsocks-rust](https://github.com/shadowsocks/shadowsocks-rust)| Rust语言编写，libev版本停止新功能开发之后的继任活跃开发版本 |
|![shadowsocks](https://img.shields.io/github/stars/shadowsocks/go-shadowsocks2)[go-shadowsocks2](https://github.com/shadowsocks/go-shadowsocks2)| Go语言编写 |

### 3. VMess / VLESS（2016年）

随着Shadowsocks的流行，GFW也逐步加强了对Shadowsocks流量的检测。**VMess** 是由 [**V2Ray**](https://zh.wikipedia.org/wiki/V2Ray) 项目引入的，V2Ray是Shadowsocks的精神继承者，代码在[github](https://github.com/v2ray)上完全开源，专为对抗GFW的封锁设计，它采用了更为复杂的加密方式，同时也提供了更高的隐蔽性。

![V2Ray](https://v2rayn.org/wp-content/uploads/2022/06/1655215063-favicon.png)

- **VMess** 协议在基础的加密之上，还加入了随机化的“伪装”机制，使得流量更难被GFW检测。
- **VLESS** 则是VMess协议的优化版本，移除了部分过于复杂的功能，简化了协议，并且增强了速度和隐蔽性。V2Ray和VLESS的结合，使得用户能在较高的隐蔽性下进行高速的翻墙。

V2Ray有一个变体[XRay](https://github.com/XTLS)，优化了性能和隐蔽性。

### 4. WireGuard（2018年）
[**WireGuard**](https://zh.wikipedia.org/wiki/WireGuard) 是一个现代化的VPN协议，代码也在[github](https://github.com/WireGuard)上完全开源，以其简洁和高效著称。WireGuard的设计重点是性能和安全性，它使用了更加先进的加密算法（如 Curve25519）。它相较于传统的VPN协议（如OpenVPN和IPSec）更加高效，且更难被审查和封锁。

![WireGuard](https://www.scalefactory.com/blog/2020/12/16/wireguard-vpn-for-remote-working/img/wireguard.png)

- **WireGuard** 协议由于其高效的传输特性和简洁的设计，使得它能够很好地对抗GFW的封锁，因此被一些翻墙软件作为一种翻墙协议采用。

### 5. Trojan / Trojan-Go（2019年）

[**Trojan**](https://github.com/trojan-gfw) 是一种基于HTTPS的翻墙协议，其设计思路是使得翻墙流量看起来与正常的HTTPS流量没有区别，从而避免被GFW识别。Trojan协议利用TLS加密，能有效对抗深度包检测（DPI）。

![trojan](https://sshs8.com/wp-content/uploads/2023/07/What-is-Trojan-GFW-VPN-and-How-Does-It-Work.png)

- [**Trojan-Go**](https://github.com/p4gefau1t/trojan-go) 是Trojan协议的改进版，支持更多的加密方式和更高效的协议切换。

### 6. NaïveProxy（2020年）

[**NaïveProxy**](https://github.com/klzgrad/naiveproxy) 通过将通信伪装成正常的HTTPS流量来实现翻墙，它的工作原理和Trojan类似，采用TLS/SSL加密技术来隐藏流量。

- 由于**NaïveProxy** 利用TLS协议，它能够极好地伪装流量，从而有效绕过GFW的封锁。

### 7. Hysteria（2020年代初）

[**Hysteria**](https://github.com/apernet/hysteria) 是一个相对较新的翻墙协议，专注于高性能和低延迟，支持UDP流量传输，非常适合实时应用（如视频流和游戏）。Hysteria协议基于[QUIC](https://zh.wikipedia.org/wiki/QUIC)协议，因此可以利用QUIC的优势来规避GFW的监控。

### 8. Mieru（2021年）

[**Mieru**](https://github.com/enfein/mieru) 是一个相对新颖的协议，它也着重于优化翻墙的性能和隐蔽性。这个协议的设计理念是类似于WireGuard和Shadowsocks，它采用了简单而强大的加密技术，且与现有的TLS/SSL协议兼容。

### 9. TUIC（2022年）

[**TUIC**](https://github.com/tuic-protocol/tuic) 是一个新的、基于QUIC协议的翻墙协议。它的设计目标是提供更高效的加密和传输速度。QUIC协议的优势在于它具有更低的延迟，并且加密强度较高，因此对抗GFW的封锁能力更强。

### 为什么会有这么多协议？

随着技术的发展和GFW封锁手段的不断升级，翻墙协议的设计逐渐从简单的代理方式转向更加复杂的加密和伪装方法。每个新的协议和工具，基本上都是应对更复杂的网络审查机制，包括深度包检测（DPI）、IP封锁、域名劫持等手段。为了突破这些防御机制，翻墙工具需要不断进行技术创新，因此产生了多种协议。

### 客户端软件

以下是常见的翻墙客户端软件及其支持的协议、发布年份、GitHub地址等信息。客户端软件通常用于连接到翻墙服务器或代理服务，并进行数据加密和转发。

| 客户端软件         | 发布年份 | 支持的协议                     | GitHub 地址 | 备注                                           |
|------------------|----------|----------------------------|-------------|----------------------------------------------|
| **Shadowsocks**     | 2012年   | Shadowsocks                  | [GitHub](https://github.com/shadowsocks/shadowsocks-qt5) | 官方的桌面版客户端，支持Windows、Mac和Linux |
| **V2RayN**          | 2017年   | VMess, VLESS, Shadowsocks   | [GitHub](https://github.com/2dust/v2rayN)  | V2Ray协议的Windows客户端，支持V2Ray等多种协议 |
| **V2RayNG**         | 2018年   | VMess, VLESS, Shadowsocks   | [GitHub](https://github.com/2dust/v2rayNG) | 安卓端V2Ray客户端，支持多协议，隐蔽性强 |
| **Clash for Windows** | 2020年   | V2Ray, Shadowsocks, HTTP, SOCKS5, Trojan, etc. | [GitHub](https://github.com/Fndroid/clash_for_windows_pkg) | 支持多协议的桌面客户端，配置灵活，常用于科学上网 |
| **Trojan-Qt5**      | 2019年   | Trojan (HTTPS)              | [GitHub](https://github.com/Trojan-Qt5/Trojan-Qt5) | Trojan协议的客户端，支持Windows和Linux |
| **Outline Client**  | 2017年   | Shadowsocks                 | [GitHub](https://github.com/Jigsaw-Code/outline-client) | Google Jigsaw开发的Shadowsocks客户端，简单易用 |
| **Lantern**         | 2012年   | Lantern P2P, SOCKS5         | [GitHub](https://github.com/getlantern/lantern) | P2P翻墙工具，提供简便的代理服务 |
| **ShadowsocksX-NG** | 2017年   | Shadowsocks, ShadowsocksR   | [GitHub](https://github.com/shadowsocks/ShadowsocksX-NG) | macOS平台的Shadowsocks客户端，功能完善 |
| **FlClash** | 2024年   | Clash, V2ray, Vless, Hysteria   | [GitHub](https://github.com/hiddify/hiddify-app) | Dart语言开发，支持Android, iOS, Windows, macOS, Linux等多平台，功能完善 |
| **hiddify** | 2024年   | Vless, Vmess, Reality, TUIC, Hysteria, Wireguard, SSH   | [GitHub](https://github.com/hiddify/hiddify-app) | Dart语言开发，支持Android, iOS, Windows, macOS, Linux等多平台，功能完善 |

### 总结

这些翻墙客户端软件支持的协议各不相同，有的专注于隐蔽性和安全性（如Trojan和V2Ray），有的则侧重于易用性和跨平台支持（如Outline和Lantern）。用户可以根据自己的需求选择合适的客户端。
