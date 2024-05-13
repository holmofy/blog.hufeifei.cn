---
title: “真”的IP真的是真的吗？
date: 2024-01-12
tags: HTTP
categories: 网络
description: x-forward-for头信息
---

## TLDR

这篇文章比较长，介绍全面。恐怕很多人读不完，先来一个**T**oo **L**ong **D**on't **R**ead的总结：

* 从Http标头中获取“真实客户端IP地址”时，请使用`X-Forwarded-For`列表中最右边的IP。

* `XFF`中最左边的IP通常被认为是“最接近客户端”和“最真实”的，但它很容易伪造。不要将它用于任何与安全相关的事情。

* 选择最右边的`XFF`IP时，请确保使用该Http标头的最后一个真实地址。

* 使用由反向代理设置的特殊“真实客户端IP”头（如`X-Real-IP`, `True-Client-IP`等）可能很好，但这取决于 a)反向代理实际如何设置它；b)如果它已经存在/欺骗，反向代理是否设置它；c)如果有反向代理，如何配置反向代理。

* 任何非反向代理专门设置的Http标头都不可信。比如，如果您不检查`X-Real-IP`直接在 Nginx 后面追加，那你可能将读取到伪造的值。

* 许多限速器都使用可欺骗的IP实现，这容易受到绕过限速器导致内存溢出攻击。

如果你在代码或基础设施的任何地方使用所谓的“Real IP”，你现在就需要去检查你是如何提取它的。

下面将详细解释这些内容，因此请继续阅读。

## 介绍

使用`X-Forwarded-For`或其他HTTP标头获取所谓的“Real IP”，目前地使用状态非常糟糕。这些HTTP标头设计不正确、不一致，结果导致被不恰当地使用。这导致各种项目中的安全漏洞，并且肯定会在将来导致更多的问题。

在研究了一段时间的限速器之后，我开始关心 IPv6 处理。我写了[一篇文章](https://adam-p.ca/blog/2022/02/ipv6-rate-limiting/)，详细介绍了IPv6限速如何导致速率限制器逃逸和内存溢出。然后，我转而担心限速器在负载均衡（或任何反向代理）后面时，如何确定要限速的IP。正如你所看到的，情况很糟糕。

但这不仅是关于限速器。如果你曾经接触过查看`X-Forwarded-For`头的代码，或者如果你使用别人的代码去获取所谓的“RealIP”，那么你绝对需要小心谨慎。这篇文章将帮助你理解为什么。

## 获得真正的客户端 IP 不会那么难，对吧？

Web 服务对其客户端的 IP 地址感兴趣的原因有很多：地理统计、地理定位、审计、限速、防止滥用、会话历史记录等。

当客户端直接连接到服务器时，服务器可以看到客户端的 IP 地址。如果客户端通过一个或多个代理（任何类型的代理：正向、反向、负载均衡器、API 网关、TLS 卸载、IP 访问控制等）进行连接，则服务器只能直接看到客户端连接使用的最终代理的 IP 地址。

为了将原始 IP 地址传递到服务器，有几个常用的HTTP标头：

* [`X-Forwarded-For`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-For)是逗号分隔的 IP 列表，每个经过的代理都会将访问者追加到该IP列表。按照这个想法，第一个IP（由第一个代理添加）是真正的客户端IP。每个后续 IP 都是路径上的另一个代理。最后一个代理的 IP 不存在（因为代理不添加自己的 IP，并且因为它直接连接到服务器，因此其 IP 无论如何都可以直接使用）。后面将经常讨论这个问题，所以它将缩写为“XFF”。

* [`Forwarded`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Forwarded)是最官方的头但似乎使用最少。我们将在下面更详细地介绍它，但它实际上只是 XFF 的一个更高级版本，它具有我们将要讨论的相同问题。

* 还有特殊的单 IP 头，如 X-Real-IP（Nginx）、CF-Connecting-IP（Cloudflare） 或 True-Client-IP（Cloudflare 和 Akamai）。我们将在下面详细讨论这些，但它们不是本文的主要重点。

## 陷阱

在讨论如何正确使用 XFF 之前，我们将讨论使用`X-Forwarded-For`可能出错的多种形式。

### HTTP标头不可信

首先，也是最重要的一点，您必须始终意识到，由不受您控制的任何代理添加（或似乎已经添加）的任何 XFF IP 都是完全不可靠的。任何代理都可以以任何它想要的方式添加、删除或修改HTTP标头。客户端也可以最初将HTTP标头设置为它想要的任何内容，以使欺骗球滚动。例如，如果您向 AWS 负载均衡器发出此请求：

```curl
curl -X POST https://my.load.balanced.domain/login -H "X-Forwarded-For: 1.2.3.4, 11.22.33.44"
```

负载均衡器后面的服务器将获得以下信息：

```
X-Forwarded-For: 1.2.3.4, 11.22.33.44, <actual client IP>
```

还有这个：

```curl
curl -X POST https://my.load.balanced.domain/login -H "X-Forwarded-For: oh, hi,,127.0.0.1,,,,"
```

会给你这个：

```
X-Forwarded-For: oh, hi,,127.0.0.1,,,,, <actual client IP>
```

正如你所看到的，目前都只是通过，这个HTTP标头前面的信息不会被改变也不会被验证。最终的实际 IP 只是附加到已经存在的内容后。

（除了`curl` 和自定义客户端之外，还有类似于ModHeader的[Chrome插件](https://chromewebstore.google.com/detail/x-forwarded-for-header/hkghghbnihliadkabmlcmcgmffllglin)可让您在浏览器请求中设置 XFF 头信息。但是，如何设置HTTP标头对我们来说并不重要，重要的是攻击者可以利用这一点。

### 多个HTTP标头

根据 [HTTP/1.1 RFC （2616）](https://datatracker.ietf.org/doc/html/rfc2616#section-4.2):

> Multiple message-header fields with the same field-name MAY be present in a message if and only if the entire field-value for that header field is defined as a comma-separated list [i.e., #(values)]. It MUST be possible to combine the multiple header fields into one “field-name: field-value” pair, without changing the semantics of the message, by appending each subsequent field-value to the first, each separated by a comma. The order in which header fields with the same field-name are received is therefore significant to the interpretation of the combined field value, and thus a proxy MUST NOT change the order of these field values when a message is forwarded.

这适用于 XFF，因为它是一个逗号分隔的列表。这可能使获取最右边（甚至最左边）的 IP 容易出错。

例如，Go语言有三种获取HTTP标头值的方法：

* [`http.Header.Get(headerName)`](https://pkg.go.dev/net/http#Header.Get)以字符串形式返回第一个标头值。
* [`http.Header.Values(headerName)`](https://pkg.go.dev/net/http#Header.Values)返回一个字符串切片（数组），其中包含`headerName`标头的所有实例的值。(在查找之前`headerName`会被规范化)
* `http.Header`是个`map[string][]string`，可以直接访问。（map的key是规范化的标头名称。这类似于使用`Values`。

所以这是攻击：

* Eve 使用两个伪造的 XFF 标头发出请求。
* 根据 RFC 要求，您的反向代理将 Eve 的真实 IP 添加到第二个 XFF 标头的末尾。
* 您调用`req.Header.Get("X-Forwarded-For")`并获取第一个标头。你把它分开，拿最右边的。
* 您选择了欺骗性 IP。你把它看作是值得信赖的。结果坏事了。

与 Go 不同，Twisted 获取[单个标头值的方法](https://github.com/twisted/twisted/blob/ebb2d360070e468981377b917e3a728ff4e6c7f6/src/twisted/web/http.py#L1068)返回最后一个值。（为什么没有标准的、通用的、公认的行为？这避免了上述攻击，但它可能会导致一个不同的（不太可能的）问题：如果你使用最右边的算法（如下所述），你需要从右边向后寻找第一个不受信任的IP。但是，如果您的一个反向代理添加了新的标头而不是附加（根据 RFC，这是一件有效的事情）怎么办？现在，您想要的 IP 在最后一个标头中无处可寻——它充满了受信任的反向代理 IP，而真正的 IP 位于 XFF 标头的先前实例中。

这里可能存在一种微妙的、假设的攻击：

* 你（至少）有两个你信任的反向代理。
* 第二个反向代理不喜欢超长的标头，因此它会创建一个新的标头，而不是在 XFF 标头太长时附加。
* 夏娃知道这一点。她想向你隐瞒她的IP。
* 夏娃在她给你的请求中恶搞了一个长长的XFF。
* 您的第一个反向代理将她的真实 IP 添加到 XFF 标头。
* 您的第二个反向代理不喜欢该标头的长度，因此它会创建一个新标头。标头值是第一个反向代理的 IP。
* 您的服务器软件获取最后一个标头，它只有一个 IP，属于您的第一个反向代理。
* 你的逻辑是做什么的？使用该 IP？因为它是私人的/受信任的，所以把它当作特殊的？恐慌是因为这个IP不可能被信任？

请注意，当我使用 AWS ALB 后面的服务器进行测试时，我发现 ALB 已经连接了 XFF 标头。所以这很好。我不知道其他反向代理是否也这样做，但我敢打赌没有真正的一致性。

最好的办法是自己合并所有 XFF 标头。

（值得询问和检查的是，确保反向代理附加到正确的标头，因为附加到错误的标头会破坏采取最正确的代理的可信度。我只检查了 AWS ALB 和 Cloudflare，他们做对了。如果有人发现做错了什么，请告诉我。

### 私有 IP

即使在完全非恶意的情况下，任何 XFF IP（尤其是最左边的 IP）也可能是私有/内部 IP 地址。如果客户端首先连接到内部代理，它可能会将客户端的私有 IP 添加到 XFF 标头中。这个地址永远不会对你有用。

### 拆分 IP

因为`X-Forwarded-For`不是官方标准，所以没有正式的规范。大多数示例显示 IP 地址以逗号空格（`, `） 分隔，但空格并不是严格要求的。（例如，HTTP/1.1 RFC 说像 XFF 这样的标头只是“逗号分隔”。我查看的大多数代码仅按逗号拆分，然后修剪值，但我发现至少有一个代码会查找逗号空间。

在测试时，在我看来，AWS ALB 在添加 IP 时使用逗号空间，但 Cloudflare 只使用逗号。

### 未加密的数据始终不可信

这应该不言而喻，但是如果您收到的是 HTTP-not-S 请求，那么任何人都可以在它们到达您之前修改标头。值得一提的是，闯入者无法搞砸“最右边”的方法（如下所述），因为他们无法搞砸从互联网到您的反向代理或服务器的最终连接的 IP。

所以只要加密你的流量，好吗？

### 其他标头（X-Client-IP，True-Client-IP）可能存在并被欺骗

一些反向代理会删除任何意外或不需要的标头，但有些（如AWS ALB）不会。因此，攻击者可以设置X-Client-IP、True-Client-IP标头，例如并直接连接到您的服务器。如果您的反向代理没有专门为您设置它们，您无需被愚弄使用它们。

### 尝试了解X-Forwarded-For

不幸的是，尝试让自己了解 XFF 也很困难。

MDN Web Docs 通常是此类内容的黄金标准，但关于 XFF 的页面根本没有提到这些风险; 它说“最右边的 IP 地址是最新代理的 IP 地址，最左边的 IP 地址是原始客户端的 IP 地址”，没有任何警告。维基百科条目要好得多：“由于很容易伪造`X-Forwarded-For`字段，因此应谨慎使用给定的信息。最右边的 IP 地址始终是连接到最后一个代理的 IP 地址，这意味着它是最可靠的信息来源。

【2022-03-09：为 MDN 文档创建了一个[issue](https://github.com/mdn/content/issues/13703)。 2022-03-19：我重写了页面，对其进行了公关，更改现已生效。您可以在此处查看[原始`Forwarded`页面的 PDF](https://adam-p.ca/misc/MDN-XFF.pdf)。现在要修复页面...】

其他来源也同样可变。有些人对标头被欺骗的可能性或私人地址的存在（[1](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/x-forwarded-headers.html)、[2](https://techcommunity.microsoft.com/t5/iis-support-blog/how-to-use-x-forwarded-for-header-to-log-actual-client-ip/ba-p/873115)、[3](https://www.geeksforgeeks.org/http-headers-x-forwarded-for/)、[4](https://developers.cloudflare.com/fundamentals/get-started/http-request-headers/)、[5](https://www.keycdn.com/blog/x-forwarded-for-cdn)）一无所知。其他人在提及风险方面做得很好（[6](https://totaluptime.com/kb/prevent-x-forwarded-for-spoofing-or-manipulation/),[7](https://docs.fastly.com/signalsciences/faq/real-client-ip-addresses/#x-forwarded-for-header-configuration),[8](https://datatracker.ietf.org/doc/html/rfc7239#section-8.1)），但有时您必须深入阅读才能获得警告。

## 避免这些坑

让我们做一些基线陈述：

* [使用专用地址空间中的 IP 作为“真正的”客户端 IP 从来都不是正确的选择](https://adam-p.ca/blog/2022/03/x-forwarded-for/#fn:4).
* 使用实际上不是 IP 地址的值从来都不是正确的选择。
* 在没有诡计的情况下，最左边的非私有、非无效的 IP 是我们最接近“真实”客户端 IP。（以下简称“最左边”）。
* 我们唯一可以信任的客户端 IP 是我们控制的（反向）代理添加的第一个客户端 IP。（以下简称“最右边的”）。
* **最左边的人通常是最“真实”的，而最右边的人是最值得信赖的**。那么你应该使用哪个IP？这取决于你要用它做什么。

如果你要做一些与安全有关的事情，你需要使用你信任的IP--最右边的IP。这里最明显的例子是限速。如果您为此使用最左边的 IP，攻击者可以在每个请求中欺骗不同的 XFF 前缀值，并完全避免受到限制。

此外，他们可能会通过强制您存储太多的单个条目来耗尽您的服务器内存——每个虚假 IP 一个条目。似乎很难相信将 IP 地址存储在内存中会导致耗尽 - 尤其是当它们存储在生存时间有限的缓存中时，但请记住：

* 攻击者不会局限于 40 亿个 IPv4 地址。他们可以使用所有数以亿计的 IPv6 地址，如果限制器对前缀不智能。
* 由于许多限制器不检查有效的 IP，因此攻击者可以使用所需的任何随机字符串。
* 另请注意，这些字符串可能很大;例如，Go 的默认标头块大小限制为 1MB。这意味着单个随机字符串“IP”可能接近 1MB。这意味着每个请求增加 1MB 的内存使用量。

对于所有攻击者和配置来说，它仍然不可行，但不应不加考虑地将其驳回。

或者，攻击者可以强制您对其他用户的 IP 地址进行速率限制/阻止。他们可以提供真实的 IP 地址，但不能提供他们的 IP 地址，您最终会被愚弄以限制其速率。（如果你使用“真实”的IP进行滥用报告，你最终可能会抱怨错误的人。

使用最右边的 IP 进行速率限制的缺点是，您可能会阻止一个代理 IP，该 IP 实际上不是滥用的来源，而只是被一堆不同的客户端使用，如果您只是使用最左边的 IP，您就会意识到这一点。是的，好吧。这似乎不太可能，而且它仍然比允许攻击者轻而易举地绕过您的速率限制器并使您的服务器崩溃要容易得多。

如果你正在做一些与安全无关的事情......认真考虑您的用例。假设您只想对您的统计数据进行 IP 地理位置查找。可能最左边的 IP 就是你想要的。您的绝大多数用户不会进行任何标头欺骗，并且随机互联网代理的地理位置对您没有好处，因此您可能会在最接近用户的 IP 上获得最佳结果。

另一方面，您可能需要考虑您期望拥有多少使用 Internet 代理的用户。可能足够少，如果你地理定位错误的东西，它不会损害你的统计数据。攻击者有没有办法通过故意歪曲你的地理统计数据来伤害你？可能不是，但花点时间认真考虑一下。

因此，在编写“GetRealClientIP（request）”函数时要小心。确保它有一个关于如何使用它的大警告注释。或者编写两个函数：`GetUntrustworthyRealClientIP(request)`和`GetTrustworthyButLessRealClientIP(request)`。这些都是可怕的名字。也许只是传递一面旗帜。无论如何，关键是要防止函数的调用者对结果的性质产生任何混淆。

使用该函数的结果时也要小心。编写代码很容易，让最左边的 IP 进行一些地理查找，然后决定您还需要进行速率限制......所以你不妨使用相同的“realClientIP”变量！哎呀。这可能是使错误的代码看起来错误的好时机。

请记住，最终代理 IP（或客户端的地址（如果直接连接）不在 XFF 标头中。为此，您需要查看您的请求连接信息。（在 Go 中的`http.Request.RemoteAddr`，许多 CGI 服务器的`REMOTE_ADDR`环境变量等）

### 算法

阅读本文时，请记住，最终的代理 IP 不在 XFF 列表中，而是`RemoteAddr`。另请注意，`RemoteAddr`它可能具有`ip:port`形式，具体取决于您的平台（就像在 Go 中一样）——当然可以确保只使用 IP 部分。

#### 第一：收集所有IP

列出所有`X-Forwarded-For`标头中的所有IP。`RemoteAddr`也是有用的。

#### 第二：确定您的安全需求是什么

默认使用最右边的方法。仅在必要时使用最左边的，并确保谨慎使用。

#### 最左边：最接近“真实IP”，但完全不可信

如果您的服务器直接连接到 Internet，则可能有 XFF 标头，也可能没有（取决于客户端是否使用代理）。如果存在 XFF 标头，请选择最左侧的 IP 地址，该地址是有效的非专用 IPv4 或 IPv6 地址。如果没有 XFF 标头，请使用`RemoteAddr`。

如果您的服务器位于一个或多个反向代理后面，请选择最左边的 XFF IP 地址，该地址是有效的非私有 IPv4 或 IPv6 地址。（如果没有 XFF 标头，则需要立即修复网络配置问题。

永远不要忘记安全隐患！

#### 最右边的：唯一值得信赖的有用IP

如果您的服务器直接连接到互联网，则 XFF 标头不可信。使用`RemoteAddr`。

如果您的服务器位于一个或多个反向代理后面，并且无法从 Internet 直接访问，则需要知道这些反向代理的 IP 地址或请求将通过的 IP 数量。我们将这些称为“受信任的代理 IP”和“受信任的代理计数”。（最好使用“受信任的代理 IP”，原因如“网络体系结构更改”部分所述。

受信任的代理 IP 或受信任的代理计数将告诉您在找到不属于某个反向代理的第一个 IP 之前，您需要检查距离 XFF 标头的右侧多远。此 IP 是由您的第一个受信任的代理添加的，因此是您唯一可以信任的 IP。使用它。

（请注意，我在这里说的不是“有效的非私有IP”。这样做很诱人，只是为了更加安全，如果你这样做，我不会责怪你，但如果你不能相信你自己的反向代理来添加适当的IP，那么你就会遇到更大的问题。

同样，如果您支持一个或多个反向代理并且没有 XFF 标头，您需要立即弄清楚人们如何直接连接到您的服务器。

##### 暂定变化：最右边的非私有IP

如果您的所有反向代理都与您的服务器位于同一私有 IP 空间中，我认为可以使用最右边的非私有 IP，而不是使用“受信任的代理 IP”或“受信任的代理计数”。这相当于将所有专用 IP 范围添加到“受信任的代理 IP”列表中。

这不起作用的一个例子是，如果您位于外部反向代理服务（如 Cloudflare）后面——它不在您的私有地址空间中。

## 掉进那些坑里

让我们看看真实世界的例子！

警告：我在这里有点得意忘形。我只打算看看几个我熟悉的项目，但危险使用最左边的命中率太高了，所以我一直在寻找。（即使做得好，也有一些有趣和有教育意义的方面。

（如果这里没有提到某个工具或服务，那是因为我没有看过它，或者找不到足够的信息。我包括了所有的成功和失败。

### Cloudflare、Nginx、Apache

让我们从一些好消息开始。

Cloudflare 将`CF-Connecting-IP`标头添加到通过它的所有请求中;它添加`True-Client-IP`为需要向后兼容性的企业用户的同义词。这些标头的值是单个 IP 地址。我能找到的对这些标头的最完整描述听起来像是它们只是使用最左边的 XFF IP，但这个例子不够完整，我自己也尝试了一下。令人高兴的是，看起来他们实际上使用了最正确的方法。

Nginx 提供了一个默认[未启用的模块](https://nginx.org/en/docs/http/ngx_http_realip_module.html)，用于添加 X-Real-IP 标头。这也是一个单一的IP。正确且完全配置时6，它还使用不在“受信任”列表中的最右边的 IP。所以，最右边的IP。也不错。

同样，当配置为 查看`X-Forwarded-For` 时，Apache 的[mod_remoteip](https://httpd.apache.org/docs/trunk/mod/mod_remoteip.html)会选择最右边的不受信任的 IP 进行设置`REMOTE_ADDR`。

### Akamai公司

Akamai 做了非常错误的事情，但至少对此发出了警告。以下是有关它如何处理`X-Forwarded-For`和`True-Client-IP`（原始强调）的[文档](https://community.akamai.com/customers/s/article/Difference-Between-Akamai-True-Client-IP-header-and-Default-X-Forwarded-For)：

> X-Forwarded-For header is the default header proxies use to report the end user IP that is requesting the content. However, this header is often overwritten by other proxies and is also overwritten by Akamai parent servers and thus are not very reliable.
>
> The True-Client-IP header sent by Akamai does not get overwritten by proxy or Akamai servers and will contain the IP of the client when sending the request to the origin.
>
> True-Client-IP is a self provisioned feature enabled in the Property Manager.
>
> Note that if the True-Client-IP header is already present in the request from the client it will not be overwritten or sent twice. It is not a security feature.
>
> The connecting IP is appended to X-Forwarded-For header by proxy server and thus it can contain multiple IPs in the list with comma as separator. True-Client-IP contains only one IP. If the the end user uses proxy server to connect to Akamai edge server, True-Client-IP is the first IP from in X-Forwarded-For header. If the end user connects to Akamai edge server directly, True-Client-IP is the connecting public IP seen by Akamai.

相关位是“`True-Client-IP`是`X-Forwarded-For`标头中的第一个 IP”和“如果 True-Client-IP 标头已经存在于来自客户端的请求中，则不会被覆盖”。因此，`True-Client-IP`要么是最左边的 XFF IP，要么是保留客户端欺骗的原始值。只是最糟糕的事情。

但是，也有一句话“这不是安全功能”。嗯，这当然是真的。这个警告可以吗？没有大量 Akamai 用户出于安全相关目的使用`True-Client-IP`的可能性有多大？

（我不确定如何解释上面的内容，当它说 XFF 标头“被 Akamai 父服务器覆盖”时。当它说“覆盖”时，它是否意味着“附加到”？还是 Akamai 实际上吹走了现有的标头值？这将违背XFF的精神。

> 原文地址：https://adam-p.ca/blog/2022/03/x-forwarded-for/
