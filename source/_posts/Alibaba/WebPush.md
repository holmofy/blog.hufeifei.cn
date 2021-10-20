---
title: AE在WebPush技术中的探索之路
date: 2020-10-15
categories: JAVA
tags: 
- WebPush
- JAVA
- Alibaba
keywords:
- WebPush
---

之前几个月在做的WebPush项目，最早接入Firebase的方案，看了一下官方的demo觉得这个需求应该还是很简单的，直到做的过程中遇到[fcm token失效](https://github.com/firebase/quickstart-js/issues/101)的各种坑，以及AE多语言站点的问题，可以说是一路坎坷。

由于国内连不上谷歌服务，而谷歌的Chrome市场占有率又是最大的，所以国内基本上很少看到有做WebPush的网站，也正因此关于WebPush的中文资料寥寥无几，可以借鉴的网站都是国外的。我们前端接了Firebase的SDK由于体积太大了，移动端也被迫下线，真是屋漏偏逢连夜雨。

在ATA上搜了一下WebPush相关技术，看到在16年就有[AE的同学做过WebPush](https://www.atatech.org/articles/64523?spm=ata.13269325.0.0.276449faRcTLzm)，从ODPS上的一张表看出18年波哥也基于Firebase做过WebPush，最后应该都无疾而终了，顿时觉得WebPush远比想象的复杂。

所以国庆回家几天，研究了一下WebPush的规范，以期能从Firebase的泥沼中得到解脱。

# 1、浅谈Push消息

Push消息是移动互联网的产物。作为移动端两大操作系统，iOS和Android有各自的消息推送服务，苹果有APNs(Apple Push Notification service)，谷歌有GCM(Google Cloud Messaging)和FCM(Firebase Cloud Messaging)，FCM是[Firebase被Google收购](https://en.wikipedia.org/wiki/Firebase)后与GCM结合的产物，移动互联网竞争失败者微软也有自己的MPNs。

APNs和GCM都是系统级的推送能力，以APNs为例，[iOS系统会有一个5223端口专门用于连接APNs服务](https://support.apple.com/en-us/HT203609)。而安卓方面，由于政策原因以及它的开源生态，导致GCM在国内几乎无法使用。一般方案都是开发者自己在App的[后台进程](https://developer.android.com/guide/components/services)中与自己的服务端建立通信。但这样方案缺点是进程被kill后，push就收不到了，所以大家都利用App的进程拉活来保证后台进程活跃，这也导致了前几年[Baidu Push](http://push.baidu.com/)、[AliPush](https://www.aliyun.com/product/cps)、腾讯的[信鸽](https://xg.qq.com/)、[友盟](https://www.umeng.com/push)、[个推](https://www.getui.com/)、[极光](https://www.jiguang.cn/)等各种推送联盟繁盛。

各种自建通道需要维持长连接[耗电耗内存耗流量](https://www.sohu.com/a/197188420_99928473)，导致Android用户体验极差，这也促使[工信部牵头建设统一推送联盟](https://www.sohu.com/a/207836945_114760)。由于利益关系，各大厂商都想主导这个系统，导致进展缓慢，于是国内各手机厂商都在建立[自己系统的Push通道](https://docs.growingio.com/mp/developers/push-channel/)。但是各家厂商又都有自己的小算盘，毕竟有较大的人力物力投入，所以通道被商业化，比如下发速度等上面做一些文章，这也就与原始Push通道体验问题慢慢变味。另一方面，如果App要有非常好的推送体验，必须集成多家厂商的推送SDK，还有三方的推送SDK，这不仅有很大的开发压力还对自身App的体积是不小的考验。

![Push消息](https://img.alicdn.com/tfs/TB1Q8.LYYY1gK0jSZTEXXXDQVXa-808-797.png)

# 2、从AppPush到WebPush

在任何操作系统上浏览器都是使用最频繁的App，随着浏览器的性能提升，很多应用都没有独立的App，而是建立在Web上。WebApp的兴起也激发了WebPush的需求。

![WebPush](https://www.datocms-assets.com/6307/1565096354-web-push-notification-history.png)

谷歌除了Android操作系统拥有市场最大占比，Chrome也在浏览器市场独占鳌头。苹果和谷歌一样都有自己的操作系统和浏览器，所以苹果的[APNs](https://developer.apple.com/notifications/safari-push-notifications/)和谷歌的FCM也最早支持了WebPush消息。

PC操作系统上，苹果的APNs在MacOS与iOS上建立了统一的推送服务。微软从Win8开始就借鉴了苹果的很多设计思路，建立了应用商店和[WNS(Windows Push Notification Services)](https://docs.microsoft.com/en-us/windows/uwp/design/shell/tiles-and-notifications/windows-push-notification-services--wns--overview)，WNS就是想统一Windows和WinPhone的消息推送，无奈WP在移动端竞争失败，而PC上Win32传统应用已经根深蒂固，[商店应用](https://en.wikipedia.org/wiki/Microsoft_Store_(digital))难以推广。不可不提的是Windows自家的Edge浏览器的推送服务就是基于WNS的。

![WebPush](https://img.alicdn.com/tfs/TB193eAiNvbeK8jSZPfXXariXXa-1683-974.png)

Mozilla的Firefox仿佛是独立于三界之外的产物，但是却推动了整个[WebPush规范](https://tools.ietf.org/wg/webpush/)的制定，Mozilla的云服务([MCS](https://blog.mozilla.org/services/))也提供了规范中的Push Service的实现——[autopush](https://github.com/mozilla-services/autopush)。目前[FCM已经支持了Push规范](https://developers.google.com/web/updates/2016/07/web-push-interop-wins)，[Edge浏览器也在2018年4月支持了Push标准](https://blogs.windows.com/msedgedev/2018/05/22/get-started-web-push-notifications-tutorial-demo/)。苹果的Safari由于其一贯的封闭式发展，目前仍然未支持Push API规范。

![](https://img.alicdn.com/tfs/TB1aj.nY.z1gK0jSZLeXXb9kVXa-1896-1202.png)

# 3、WebPush规范

WebPush规范分为三部分：WebPush的推送方式([RFC8030](https://tools.ietf.org/html/rfc8030))，WebPush消息的加密([RFC8291](https://tools.ietf.org/html/rfc8291))，客户端浏览器与应用服务的识别([RFC8292](https://tools.ietf.org/html/rfc8292))。

## 3.1、WebPush工作方式

[RFC8030](https://tools.ietf.org/html/rfc8030)中将Push服务分为三个角色：

* UA就是用户浏览器；
* Push Service就是[FCM](https://firebase.google.com/docs/cloud-messaging)、[Mozilla Push Service](https://mozilla-push-service.readthedocs.io/en/latest/)、MNS等云服务；
* App Server就是我们自己开发的应用。

![WebPush工作方式](https://img.alicdn.com/tfs/TB1gGZwY.Y1gK0jSZFMXXaWcVXa-996-604.png)

整个过程分为一下几步：

1、用户在浏览器中点击接受订阅后，会向Push Service发起订阅请求；

2、Push Service会为用户的浏览器分配一个订阅信息；

3、开发者需要在拿到订阅信息后上传到自己的App Server上；

4、后面应用服务器就可以使用浏览器的订阅信息要求Push Service向用户推送相应的Push消息了。

看上去很简单的步骤，这里面其实涉及了很多问题：

* Push Service会与若干个浏览器建立非常多的TCP连接，高效的管理这些连接对Push Service是个挑战。好在现在I/O multiplex技术很成熟，Java中也已经有Netty这样的高性能网络库了。

* App Server向Push Service推送消息后，如果用户浏览器没打开，需要等到用户浏览器打开才能收到消息。消息的实时性可能收到影响，这个时候可能需要设置一下消息的过期时间(TTL)。

* [浏览器订阅信息的过期问题](https://developer.mozilla.org/en-US/docs/Web/API/PushSubscription/expirationTime)，Push规范提供了这个字段，但是[实际多久过期取决于Push Service](https://blog.pushpad.xyz/2018/09/web-push-subscription-age-affects-delivery-rates/)，WebPush标准没有强制订阅信息到期。

  > 这里有统计各浏览器订阅信息时长的统计：https://blog.pushpad.xyz/2018/09/web-push-subscription-age-affects-delivery-rates/

* Push消息的安全性，首先要求应用站点必须基于HTTPS协议，另外为了保证中间人Push Service拿到消息后无法将消息传给第三方或者篡改消息，在[RFC8291](https://tools.ietf.org/html/rfc8291)中定义了WebPush消息加密的标准。

* App Server如何向Push Service标识自己，从而保证拿到用户订阅信息后只有App Server能发送给制定的浏览器，其他人通过该订阅信息无法发送消息给用户。[RFC8292](https://tools.ietf.org/html/rfc8292)中就定义了应用服务器识别协议VAPID(Voluntary Application Server Identification)。有了VAPID，Chrome上就不再需要遵循FCM的推送步骤，也[不需要在Firebase中创建项目拿到`gcm_sender_id`了](https://developers.google.com/web/updates/2016/07/web-push-interop-wins#introducing_vapid_for_server_identification)。

## 3.2、VAPID协议与消息加密协议

对开发者来说，VAPID协议是WebPush的关键，因为这个规范定义了App Server与Push Service的握手以及确定Push Service往哪个客户端发送消息。

这个过程其实也很简单，前提是你对公钥加密算法比较熟悉：

1、首先需要创建一对公/私钥，公钥由运行在浏览器端的Web站点持有，私钥由我们的App Server持有。

2、当用户在浏览器点击接受订阅后，Web程序需要将公钥作为options参数传入到[PushManager.subscribe()](https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe)方法中。

3、调用subscribe方法后，浏览器会向Push Service发送订阅请求注册该浏览器。请求中包括Web程序的公钥，Push Service会返回一个Token用于标识该浏览器。下图是Chrome向FCM注册设备时的Http报文：

![subscribe](https://img.alicdn.com/tfs/TB1SoMCY.z1gK0jSZLeXXb9kVXa-2578-1384.png)

4、拿到这个token，我们就可以拼接出一个http endpoint向push service发送消息了。

比如上面的token就可以拼接出FCM的endpoint：

```curl
https://fcm.googleapis.com/fcm/send/fDWFicUHqAw:APA91bFuV2sz0Uz55b1WkLdfXMcUDlA9w7dZZJyvn1XMTPhW0rqMND83gjQ_s6gP60xZn-ubnroyCxJpr4Ejkk-H_F23H0pCwOpyKjEDp3QZdPJlNM1-RmOxWtcOFGjSI5MiKE3F4Qtn
```

5、同时浏览器还会生成一对用于加密Push消息的公/私钥。

[Subscription.getKey(name)](https://developer.mozilla.org/en-US/docs/Web/API/PushSubscription/getKey)方法可以获取两个Key：

```javascript
// p256h是客户端生成的公钥，私钥会保存在浏览器中。
// 这样能保证App Server加密后的数据只能被相应的浏览器揭秘
// 而不会被中间人Push Service解密
var key = subscription.getKey('p256dh');
// 客户端还会生成一个用于加密的认证信息
var auth = subscription.getKey('auth');
```

6、服务端存储了这endpoint、publicKey、auth三个信息后，就可以给客户端推送push了。这里AppServer的加密方式和HTTPS的有点类似，它不是使用客户端的PublicKey直接对Push消息加密，而是根据客户端的PublicKey生成一个类似于HTTPS会话密钥的对称密钥，再使用[AES/GCM](https://en.wikipedia.org/wiki/Galois/Counter_Mode)对称加密算法对消息体进行加密。这样结合了对称加密的高速与非对称加密的安全。具体加密机制可以参考[RFC8291](https://tools.ietf.org/html/rfc8291)。

## 3.3、WebPushLib

要自己实现VAPID和消息加密的这一套流程，那肯定得费九牛二虎之力的(也确实有前人自己实现了，看[这里](https://golb.hplar.ch/2019/08/webpush-java.html))。不过比较幸运是[WebPushLib](https://github.com/web-push-libs)早在15年使用Javascript实现了这一套算法，后面陆续也有了其他语言的实现。我觉得16年AE的那个前辈没有成功的原因就是没有这么一个java库（[因为Java的这个库是16年5月30号才开始开发的](https://github.com/web-push-libs/webpush-java/issues?q=is%3Aissue+is%3Aclosed+sort%3Acreated-asc)）。

有了WebPushLib一切就如拨云见日，但是AE还是有一个问题亟待解决。

# 4、AE的多站点授权问题

AE由于国际化的特性，每个站点都会有不同域名，比如主战是`www.aliexpress.com`，俄罗斯站点域名是`aliexpress.ru`，德国站点是`de.aliexpress.com`...。除此之外，AE业务很复杂，根据业务还会有`my.aliexpress.com`，`trade.aliexpress.com`，`sale.aliexpress.com`，`best.aliexpress.com`等域名。

Web Push Permission的权限是基于域名的，每个域名都会有自己的权限，这就可能导致同一用户会有多个订阅信息。推送时可能会给同一个用户推送重复的消息。

[谷歌了一下multi origin web push](https://www.google.com/search?q=multi+origin+web+push)，谷歌官方都说没有完美的解决方案。唯一提到一个解决方案是根据浏设备信息(屏幕大小等数据)生成一个id，用作标识。这个我们前期也用了[fingerprint](https://github.com/fingerprintjs/fingerprintjs)生成了一个browserId，但是第一次用的时候囊括了所有的数据包括浏览器的插件和版本信息，导致browserId不稳定。

![Multi Origin web push](https://img.alicdn.com/tfs/TB1j.7LY1L2gK0jSZPhXXahvXXa-2816-1288.png)

后面受[daraz](https://www.daraz.pk/)的灵感，觉得可以通过二次弹窗跳到统一的域名进行授权。[这个网站](https://www.foxpush.com/demos.html)有几个Demo可以展示这种做法。

![subscribe](https://img.alicdn.com/tfs/TB1HvZ4o8Bh1e4jSZFhXXcC9VXa-2582-1464.png)

这种方案将授权收口到一个域名上，好处显而易见，不用维护多套域名的授权信息，极大的避免了web push消息重复推送的问题。

但是还有一个主要矛盾，每个域名站点怎么确定它是否需要弹窗进行跳转。比如`www.xxx.com`域名跳转到了订阅页面，后面`de.xxx.com`、`ru.xxx.com`域名就不应该弹窗跳转。这貌似又进入了一个死胡同，还是需要browserId的唯一标识对用户浏览器进行识别。

有一个妥协产品方案是，让每个域名都弹窗一次，弹完后在cookie中记录个数据，保证下次不弹窗。这种方案的弊端是用户真的在`www.xxx.com`、`de.xxx.com`、`ru.xxx.com`站点间切换，第一次进入的时候每个站点都会弹窗一次。另外用户如果把`www.xxx.com`站点cookie清除了，后面还是会弹窗。

已经绞尽脑汁儿了，看上去简直不能再有别的方案了。

最后受[Complex Web Push Integrations](https://documentation.onesignal.com/docs/web-push-complex-integrations)文章和[StackOverflow](https://stackoverflow.com/questions/40240155/how-subscribe-pushmanager-via-iframe-in-chrome/40256223#40256223)这篇回答的启发，捣鼓了一个iframe的方案可以完美解决弹窗问题。

目前我们的主要矛盾是，`www.xxx.com`、`de.xxx.com`、`ru.xxx.com`每个站点都会弹跳转弹窗，所以如果这几个站点都能访问到`subscribe.xxx.com`存在浏览器中的授权数据就能确定自己是否需要弹窗了。

我们可以在`www.xxx.com`、`de.xxx.com`、`ru.xxx.com`这几个站点内嵌一个不可见的iframe，iframe内负责提供`subscribe.xxx.com`域名下授权信息，主站通过访问iframe的[contentWindow](https://developer.mozilla.org/en-US/docs/Web/API/HTMLIFrameElement/contentWindow)对象拿到授权数据。有个问题是iframe是不允许跨站点访问的，好在HTML5中提供了[Window.postMessage()](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage)API支持window间的跨站点通信。

![postMessage](https://img.alicdn.com/tfs/TB1Jw_dpkcx_u4jSZFlXXXnUFXa-1492-886.png)



**Refs：**

浅谈Push消息推送：https://www.atatech.org/articles/173453

WebPush相关规范：https://tools.ietf.org/wg/webpush/

web-push-basic-knowledge：https://pushpushgo.com/en/blog/web-push-basic-knowledge/

web-push-notifications-history-effectiveness-more：https://blog.resellerclub.com/web-push-notifications-history-effectiveness-more/

get-started-web-push-notifications-tutorial-demo：https://blogs.windows.com/msedgedev/2018/05/22/get-started-web-push-notifications-tutorial-demo/

Web Push Notification Demo From Chrome：https://developers.google.com/web/fundamentals/codelabs/push-notifications

web-push-protocol：https://developers.google.com/web/fundamentals/push-notifications/web-push-protocol

Web Push Notifications Demo From Edge：https://webpushdemo.azurewebsites.net/

Send WebPush by Firebase：https://golb.hplar.ch/2018/01/Sending-Web-push-messages-from-Spring-Boot-to-Browsers.html

Sending Web Push Notifications with Java：https://golb.hplar.ch/2019/08/webpush-java.html

sending-vapid-identified-webpush-notifications-via-mozillas-push-service：https://blog.mozilla.org/services/2016/08/23/sending-vapid-identified-webpush-notifications-via-mozillas-push-service/

web-push-interop-wins：https://developers.google.com/web/updates/2016/07/web-push-interop-wins

how-to-reduce-unsubscribe-rate-push-notification：https://blog.pushengage.com/how-to-reduce-unsubscribe-rate-push-notification/

web-push-subscription-age-affects-delivery-rates：https://blog.pushpad.xyz/2018/09/web-push-subscription-age-affects-delivery-rates/

web-push-complex-integrations：https://documentation.onesignal.com/docs/web-push-complex-integrations

cross-document-communication-with-iframes：https://benohead.com/blog/2015/12/07/cross-document-communication-with-iframes/
