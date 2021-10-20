---
title:  编码全解：从ASCII/ISO-8859/GB2312/GBK到Unicode的UCS-2/UCS-4/UTF-8/UTF-16/UTF-32
date: 2017-06-02
categories: 计算机组成
tags:
- CS
- Encoding
keywords:
- 字符集编码
- ASCII
- GBK
- UTF-8
- UTF-16
---

# 1、ASCII编码

为了能在电报、打印机、计算机等电信设备上进行信息交换，就必须为不同的设备制定统一的编码格式。早期的电信设备字符编码基本都是使用6位编码。1963年美国国家标准协会(ANSI)制定并公布的ASCII编码是第一个被广泛采用7位编码。

[ASCII](https://en.wikipedia.org/wiki/ASCII)全称：American Standard Code for Information Interchange，美国信息交换标准码。直至1986年最后一次修订，ASCII共定义了128个字符：

![ASCII编码速查表](http://tva1.sinaimg.cn/large/bda5cd74ly1g0wvfjtf7yj21g011swhx.jpg)

## 1.1、控制字符

其中0x00~0x1F这32个属于[控制字符](https://en.wikipedia.org/wiki/Control_character)，这些控制字符用于控制通信设备，比如0x07(BEL，bell)表示蜂鸣器响铃，0x08(BS，backspace)表示退格，0x0A(LF，line feed)表示换行，0x0D(carriage return)表示回车，0x0C(FF，form feed)表示换页...

回车和换行被分为两个字符是有一定的历史原因的。早期的打字机回车、换行是两个动作。回车就是打字机把纸张重置到行首位置，换行就是把纸张上移一行。
![老式打字机](http://tva1.sinaimg.cn/large/bda5cd74ly1g0wvg5fjrfj20jg0gqh0l.jpg)

这些控制字符在打印设备与字符屏幕等终端上无法显示，一些终端设备为这些控制字符提供了拓展，使这些字符显示笑脸、扑克牌花色等符号。

## 1.2、打印字符

0x20~0x7F为可显示字符，其中比较特殊的`0x20`第32号字符为空格字符，`0x7F`第127号字符为DEL控制字符。

在很多编程语言中会把空格与控制字符作为空白字符处理。所以实际可显示图形字符共94个。

> 94这个特殊的数字对于在后面GB2312中会再次出现。

![控制字符的拓展](http://tva1.sinaimg.cn/large/bda5cd74ly1g0wvghyn7kj20r00hlds7.jpg)

## 1.3、ASCII编码的国际化

美国的很多国家标准都被ISO组织给国际化了，ASCII也不例外。

国际标准化组织和国际电工委员会ISO/IEC于1972年制订了[ISO/IEC 646](https://en.wikipedia.org/wiki/ISO/IEC_646)标准，它来自于多个国家标准(主要是ASCII)，允许其他国家根据需要修改ASCII中的`$` `@` `[` `\` `]` `^` `{` `|` `}` `~`12个字符为自己国家使用。

# 2、EASCII编码

欧洲国家与美国走得最近的，美国在计算机方面的发展也影响到了欧洲国家，编码问题就是其中之一。

很多欧洲国家觉得ASCII码中的95个可显示字符并不能表示本国语言的所有字符。而ASCII码使用7位编码，第8位作为校验纠错位，对于计算机内存而言，校验纠错变得不是那么必要了，使用8位编排256个字符明显不需要额外的编程成本和存储成本。于是许多的制造商在ASCII的基础上添加了128个扩展字符，其中比较流行的是IBM和微软共同制定并在MS-DOS上使用的[CP437](https://en.wikipedia.org/wiki/Code_page_437)编码（Code Page 437）。

![扩展ASCII编码](http://tva1.sinaimg.cn/large/bda5cd74ly1g0wvgv8rutj20fx08ytb1.jpg)

# 3、ISO-8859编码

由于各国制造厂商的EASCII扩展编码不统一，于是ISO/IEC委员会颁布了8位字符编码标准——[ISO/IEC 8859](https://en.wikipedia.org/wiki/ISO/IEC_8859)。

这个编码标准共有16部分，分别以`ISO/IEC 8859-n`表示(其中ISO/IEC 8859-12印度梵文已废弃)，这些字符集可以以`ISO-8859-n`作为MIME名称，比如最常见的iso-8895-1是西欧语言的8位编码标准。

下面是ISO-8895的16个编码标准：

|       部分        |       名称       | 颁布时间/最后修订时间 |
| :-------------: | :------------: | :---------: |
| ISO/IEC 8859-1  |    拉丁语系1-西欧    |  1987/1998  |
| ISO/IEC 8859-2  |    拉丁语系2-中欧    |  1987/1999  |
| ISO/IEC 8859-3  |    拉丁语系3-南欧    |  1988/1999  |
| ISO/IEC 8859-4  |    拉丁语系4-北欧    |  1987/1998  |
| ISO/IEC 8859-5  | 斯拉夫语(Cyrillic) |  1988/1999  |
| ISO/IEC 8859-6  |  阿拉伯语(Arabic)  |  1987/1999  |
| ISO/IEC 8859-7  |   希腊语(Greek)   |  1987/2003  |
| ISO/IEC 8859-8  |  希伯来语(Hebrew)  |  1988/1999  |
| ISO/IEC 8859-9  |   拉丁语系5-土耳其语   |  1989/1999  |
| ISO/IEC 8859-10 |  拉丁语系6-北日耳曼语   |  1992/1998  |
| ISO/IEC 8859-11 |    泰语(Thai)    |    2001     |
| ISO/IEC 8859-13 |   拉丁语系7-波罗的语   |    1998     |
| ISO/IEC 8859-14 |   拉丁语系8-塞尔特语   |    1998     |
| ISO/IEC 8859-15 |    拉丁语系9-西欧    |    1999     |
| ISO/IEC 8859-16 |   拉丁语系10-东南欧   |    2001     |
| ISO/IEC 8859-12 | 原预留给印度梵文,后搁置废弃 |    搁置废弃     |

# 4、汉字编码GB2312

信息技术飞速发展，计算机也开始进入中国，但是有个很严峻的问题：中华文化博大精深，光常用汉字就有几千个，这明显无法用单个字节进行编码，于是智慧的中国人民使用两个字节搞出了GB2312编码，并美其名曰：[DBCS](https://en.wikipedia.org/wiki/DBCS)（Double Byte Charecter Set 双字节字符集）。

## 4.1、GB2312介绍

GB2312，全称信息交换用汉字编码字符集，代号：GB 2312—1980，GB是“国标”汉字拼音的首字母，1980说明是1980年发布的。GB2312编码适用于汉字处理、汉字通信等系统之间的信息交换，通行于中国大陆，它所收录的汉字已经覆盖中国大陆99.75%的使用频率；新加坡等地也采用此编码。中国大陆几乎所有的中文系统和国际化的软件都支持GB 2312。台湾、香港、澳门等使用繁体字的地区则使用台湾公司开发的[BIG5](https://en.wikipedia.org/wiki/Big5)编码。

## 4.2、GB2312是如何编码的

前面提到ASCII码中`0x00~0x20`是控制字符的范围，`0x20~0x7F`是打印字符的范围，而且`0x20`是空格符，`0x7F`是DEL控制字符，所以实际可显示字符去掉一个头一个尾剩下0x21~0x7E共94个字符。

![ASCII分区](http://tva1.sinaimg.cn/large/bda5cd74ly1g0ww9b2jygj20ho0ayq3b.jpg)

GB2312双字节编码与之对应：`0x80~0x9F`是预留的控制字符区，`0xA0~0xFF`中`0xA0`和`0xFF`不可用，实际可编码区为`0xA1~0xFE`，共94个可编码区。两个字节也就是94×94。

> ”0xA0~0xFF不可用“是为了遵守当时的[ISO-2022](https://en.wikipedia.org/wiki/ISO/IEC_2022)国际标准。

![GB2312编码区](http://tva1.sinaimg.cn/large/bda5cd74ly1g0wwcbai9qj20ho0ayweo.jpg)

GB2312编码空间为94 x 94，即有94个区，每个区有94个位，这些区分别是记录了那些字符呢：

1、01-09区收录除汉字外的682个字符。

如汉字标点、汉字编号、日文平假名片假名、拼音符号、阿拉伯数字、英文字母、希腊字母。

注意：这里的英文字母、阿拉伯数字、希腊字母和ASCII中的字符不是同一个东西，这些字符占用两个字节，也就是我们常说的全角字符，而ASCII码中的字母数字只占用一个字节所以叫半角字符。

2、10-15区为空白区，没有使用，留作扩展。

3、16-55区收录3755个一级汉字，也就是最常使用的汉字，按拼音排序，从“啊”到“座”。

4、56-87区收录3008个二级汉字，使用频率相对较少的汉字，按部首/笔画排序，从“亍”到“齄”。

5、88-94区为空白区，没有使用，留作扩展。

|  区号   | 第一个字节范围 | 第二个字节范围:A1~FE | 实际字符数 |
| :---: | :-----: | :-----------: | :---: |
| 01-09 |  A1~A9  |    部分位置未编码    |  682  |
| 10-15 |  AA~AF  |      空白       |   0   |
| 16~55 |  B0~D7  |    部分位置未编码    | 3755  |
| 56~87 |  D8~F7  |    部分位置未编码    | 3088  |
| 88~94 |  F8~FE  |      空白       |   0   |

画成图表示：

![GB2312编码分布](http://tva1.sinaimg.cn/large/bda5cd74ly1g0wwemvet6j20fa0faagj.jpg)

第一区从A1A1开始编码，第一个字符是全角空格符。

![GB2312第一区](http://tva1.sinaimg.cn/large/bda5cd74ly1g0wwf30w5ej20nf06p0t6.jpg)

下图是GB2312的16区中的汉字编码，第一个汉字是“啊”。

![GB2312的16区](http://tva1.sinaimg.cn/large/bda5cd74ly1g0wwffiuiyj20ni06n3z8.jpg)

> 点击这里查看[GB2312的编码表](http://www.qqxiuzi.cn/zh/hanzi-gb2312-bianma.php)。

# 5、对GB2312的扩展—GBK

GB2312已经覆盖了99.75%的使用频率，但是很多人名地名无法用GB2312表示，比如前总理的“镕”字，歌手陶喆的“喆”字，我老家这边有个地名叫“喆桥”。没有这些字上户口办身份证都成问题了，于是国家把GB2312进行了扩展，所以就有了GBK编码。K就是“扩展”的拼音首字母。

## 5.1、GBK介绍

GBK全称汉字内码扩展规范，是1995年制定的。

GBK编码范围为：8140－FEFE，剔除xx7F码位，共23940个码位。

共收录汉字和图形符号21886个，其中汉字（包括部首和构件）21003个，图形符号883个。

GBK编码支持国际标准[ISO/IEC10646-1](https://en.wikipedia.org/wiki/Universal_Coded_Character_Set)和国家标准[GB13000-1](https://baike.baidu.com/item/GB13000)中的全部中日韩汉字，并包含了BIG5编码中的所有汉字。

虽然GBK收录GB 13000.1-93的全部字符，但GBK是一种编码方式并向下兼容GB2312；而GB 13000.1-93等同于Unicode 1.1是一种字符集，它的几种编码方式如UTF8、UTF16LE等，与GBK完全不兼容。

> GBK是对GB2312-80的扩展，最早实现于Windows 95简体中文版。
>
> GB13000是一个等同于Unicode1.1的字符集，而非编码实现，更多相关内容看下面的统一编码字符集

## 5.2、GBK编码方式

|  区位   | 第一个字节 |    第二个字节    |  编码数   | 实际字符数  |
| :---: | :---: | :---------: | :----: | :----: |
| GBK/1 | A1~A9 |    A1~FE    |  846   |  717   |
| GBK/2 | B0~F7 |    A1~FE    |  6768  |  6763  |
| GBK/3 | 81~A0 | 40~FE(7F除外) |  6080  |  6080  |
| GBK/4 | AA~FE | 40~A0(7F除外) |  8160  |  8160  |
| GBK/5 | A8~A9 | 40~A0(7F除外) |  192   |  166   |
| 用户定义区 | AA~AF |    A1~FE    |  564   |        |
| 用户定义区 | F8~FE |    A1~FE    |  658   |        |
| 用户定义区 | A1~A7 | 40~A0(7F除外) |  672   |        |
|  合计   |       |             | 23,940 | 21,886 |


* GBK/1区实际上就是原来GB2312的01~09区，并添加了度量单位符号，以及一些特殊图形
* GBK/2区合并了原来GB2312的一级汉字和二级汉字，也就是原来的16~55，56~87区
* GBK/3是扩增的新区，第一个字节为81~A0，第二个字节可以是40~FE(除去7F)，包括大量繁体字
* GBK/4是扩增的新区，第一个字节为AA~FE，第二个字节可以是40~A0(除去7F)，包括大量繁体字
* GBK/5是对原来的GB2312的符号区的扩展。
* 三个用户定义区，包括原来GB2312的两个空白区，再加一个A1~A7的空白区。

![GBK编码分布](http://tva1.sinaimg.cn/large/bda5cd74ly1g0wwo37aasj20i30hb76y.jpg)

> 点击这里查看[GBK编码表](http://www.qqxiuzi.cn/zh/hanzi-gbk-bianma.php)。
>
> 事实上最新的中文编码国家标准是[GB18030](http://baike.baidu.com/item/gb18030)，里面收录了7万多个汉字符号，但由于Unicode的普及，该编码方式已经很少被采用

## 5.3、验证中文编码

我们在Windows下的记事本中写下`啊　亍`。“啊”是GB2312第一个中文字符，中间全角空格是GB2312是第一个编码字符，“亍”是二级汉字中的第一个字符。用GB2312保存，理论上应该会得到`B0A1 A1A1 D8A1`

![保存为GB2312编码](http://tva1.sinaimg.cn/large/bda5cd74ly1g0wwp81u7kj209l06jdg3.jpg)

选择ANSI保存中文。[ANSI](https://baike.baidu.com/item/ANSI)代表当前国家或地区编码实现，不同国家的ANSI编码之间互不兼容。

![这里的以ANSI保存中文就是以GB2312编码保存](http://tva1.sinaimg.cn/large/bda5cd74ly1g0wwpzyio9j20ha0en0uj.jpg)

以16进制编辑器打开这个文本：

![GB2312十六进制编码](http://tva1.sinaimg.cn/large/bda5cd74ly1g0wwsnllcmj207q02xaa0.jpg)

# 6、统一编码字符集

## 6.1、ISO 10646标准与Unicode联盟

因为当时每个国家都有自己的一套编码规范，比如大陆有GB2312，台湾有BIG5，欧洲各国有它们相应的ISO-8859编码，日本有JIS，总之这个国家的文件到另外一个国家的电脑上打开就一大堆乱码，跨国公司可能要崩溃了。

1984年，ISO组织决定把各国编码统一起来，造一个“万国码”通行世界，1990年第一份ISO 10646标准就出炉了，但我们中国看到这个标准就很不开心了，因为韩国、日本这些中国周边的国家历来被中华文化熏陶，它们的很多历史文献、日常用字中也包含汉字，所以也有一套自己的中文编码，ISO 10646第一版将韩国、日本等国的汉字编码原封不动的加入到了这个标准，这就导致16位的编码中，已经无法加入中国的现有汉字编码了，只能使用32位表示，最重要的是对于中国、日本、韩国使用的同一个字符，ISO为他设计三个不一样的编码，这明显不利于各国字符编码的统一，还是会出现前面说的乱码问题。

而另一方面，1987年，施乐公司的Joe Becker和苹果公司的Lee Collins开发了统合处理全世界所有文字的统一码。1989年发表了统一码概要，基本为16位。于是中、日、韩文字([CJK](https://en.wikipedia.org/wiki/CJK_characters))统合了。基本方针为以16位处理所有文字。 1990年，完成了基于此方针的最终草案。隔年1991年1月，大致同意此方案的企业成立了统一码联盟，这里面就包括中国的华为公司。

1991年，各国希望能以一致的方式处理文字，如统一码这般，因而否决了ISO/IEC 10646的初版草案。基于中国与统一码联盟的提议，ISO协会妥协了，最终和统一码联盟成立了中日韩联合研究小组。中日韩联合研究小组将基于各国的汉字编码，独自定义定规范、制作ISO 10646和统一码的统一汉字编码。年尾，完成了Unified Repertoire and Ordering（URO）。1992年，URO就加入了ISO 10646第二版标准中。

> 点击[这里](http://www.unicode.org/standard/WhatIsUnicode.html)查看Unicode官方介绍
>
> 点击[这里](http://www.unicode.org/history/earlyyears.html)查看Unicode早年历史
>
> 点击[这里](http://www.unicode.org/charts/)查看Unicode编码表

## 6.2、UCS-2 与 UCS-4

Unicode 最初制定的基本方针是以16位处理所有的文字，早期这种编码就称为Unicode编码，由于后来Unicode又出现了UTF系列编码方案，为了区分就把16位定长编码称为UCS-2。对于原来的ASCII码字符，保持其编码值不变，只需将长度扩充为两个字节，而其他国家语言的字符则统一起来并重新进行编码，其中我们中国汉字占的地方最大，当然这里面也包括了日本韩国使用的汉字。

> 可以点击[这里](http://www.unicode.org/charts/PDF/U4E00.pdf)查看Unicode中文字符编码表。
>
> [这篇文章](http://www.cnblogs.com/sliencer/p/4000513.html)讲了Unicode中汉字的相关编码问题

但是2个字节最多能编码65536个字符，而中文简体和繁体加起来就有六万多字(包括后面扩展进去的[扩展区](https://zh.wikipedia.org/wiki/Wikipedia:Unicode%E6%89%A9%E5%B1%95%E6%B1%89%E5%AD%97))，这让其他国家怎么活，于是Unicode还准备了4字节定长编码方案——UCS-4。UCS-4使用32位编码，理论上能编码40多亿的字符，这个肯定能满足全球所有语言的文字了，估计整个银河系统一都用不完。

## 6.3、Unicode平面

Unicode中每个字符对应的编码值被叫做**Code Point**(翻译成“码点”或“码位”)。

另外还有一个概念叫做[**Plane**(平面)](https://en.wikipedia.org/wiki/Plane_(Unicode))，一个平面由65536($2^{16}$)个连续的码点组成，比如0x0000~0xFFFF被称为**基本多语言平面**(BMP)，这个平面编码的字符基本满足日常需求，而UCS-2也只能表示BMP中的字符。

下图便是BMP平面中字符分布图：

![BMP字符分布](http://tva1.sinaimg.cn/large/bda5cd74ly1g0wx30do5dj20l20eraca.jpg)

> 中间粉红色以及紫红色区域都是我们的汉字

除了BMP外还有很多其他平面，共17个平面，所以码点最大可达到0x10FFFF。

<table><tbody><tr><th>平面</th><th>始末字符值</th><th>中文名称</th><th>英文名称</th></tr><tr><td>0号平面</td><td>U+0000 ~ U+FFFF</td><td><b>基本多文种平面</b></td><td>Basic Multilingual Plane，简称<b>BMP</b></td></tr><tr><td>1号平面</td><td>U+10000 ~ U+1FFFF</td><td><b>多文种补充平面</b></td><td>Supplementary Multilingual Plane，简称<b>SMP</b></td></tr><tr><td>2号平面</td><td>U+20000 ~ U+2FFFF</td><td><b>表意文字补充平面</b></td><td>Supplementary Ideographic Plane，简称<b>SIP</b></td></tr><tr><td>3号平面</td><td>U+30000 ~ U+3FFFF</td><td><b>表意文字第三平面</b>（未正式使用）</td><td>Tertiary Ideographic Plane，简称<b>TIP</b></td></tr><tr><td>4号平面至13号平面</td><td>U+40000 ~ U+DFFFF</td><td>（尚未使用）</td><td></td></tr><tr><td>14号平面</td><td>U+E0000 ~ U+EFFFF</td><td><b>特别用途补充平面</b></td><td>Supplementary Special-purpose Plane，简称<b>SSP</b></td></tr><tr><td>15号平面</td><td>U+F0000 ~ U+FFFFF</td><td>保留作为<b>私人使用区（A区）</b></td><td>Private Use Area-A，简称<b>PUA-A</b></td></tr><tr><td>16号平面</td><td>U+100000 ~ U+10FFFF</td><td>保留作为<b>私人使用区（B区）</b></td><td>Private Use Area-B，简称<b>PUA-B</b></td></tr></tbody></table>

![第一辅助平面SMP](http://tva1.sinaimg.cn/large/bda5cd74ly1g0wx4g95afj20ks0ea40d.jpg)
![第二辅助平面SIP](http://tva1.sinaimg.cn/large/bda5cd74ly1g0wx4qe4hxj20js0erjsx.jpg)

# 7、UTF编码体系

随着计算机一起发展起来的还有互联网，那时候网络传输价格还是蛮高的，如果使用UCS-2或UCS-4对文本数据进行网络传输，那将消耗很大的带宽，特别是对于使用ASCII码或ISO-8895的美国和欧洲国家，原本可以用1个字节表示的字符，非要用两个字节甚至四个字节，这宽带费蹭蹭地往上涨啊。于是ISO组织和Unicode联盟共同开发了面向传输的Unicode编码方案——UTF(Unicode Transformation Format)。这个编码体系有UTF-1、UTF-7、UTF-8、UTF-16、UTF-32。其中被广泛接受的是UTF-8和UTF-16以及UTF-32。

## 7.1、UTF-8

UTF-8是一种变长Unicode编码方式：

<table><tbody><tr><th><font>字节数</font></th><th><font>代码点的位数</font></th><th><font>最小码点</font></th><th><font>最大码点</font></th><th><font>字节1</font></th><th><font>字节2</font></th><th><font>字节3</font></th><th><font>字节4</font></th></tr><tr><td><font>1</font></td><td><font>7</font></td><td><font>U + 0000</font></td><td><font>U + 007F</font></td><td><code>0XXXXXXX</code></td><td></td><td></td><td></td></tr><tr><td><font>2</font></td><td><font>11</font></td><td><font>U + 0080</font></td><td><font>U + 07FF</font></td><td><code>110XXXXX</code></td><td><code>10XXXXXX</code></td><td></td><td></td></tr><tr><td><font>3</font></td><td><font>16</font></td><td><font>U + 0800</font></td><td><font>U + FFFF</font></td><td><code>1110XXXX</code></td><td><code>10XXXXXX</code></td><td><code>10XXXXXX</code></td><td></td></tr><tr><td><font>4</font></td><td><font>21</font></td><td><font>U + 10000</font></td><td><font>U + 10FFFF</font></td><td><code>11110XXX</code></td><td><code>10XXXXXX</code></td><td><code>10XXXXXX</code></td><td><code>10XXXXXX</code></td></tr></tbody></table>

* 对于任意字节，如果第一位为0，则该字节表示单独的一个ASCII字符
* 对于任意字节，如果第一位为1，第二位为0，则该字节是多字节字符中的某个字节
* 对于任意字节，如果前两位为1，第三位为0，则该字节是两个字节表示的字符的第一个字节。
* 对于任意字节，如果前三位为1，第四位为0，则该字节是三个字节表示的字符的第一个字节。
* 对于任意字节，如果前四位为1，第五位为0，则该字节是四个字节表示的字符的第一个字节。

总结起来就是：一个字符的第一个字节前有多少个连续的1，这个字符就需要多少个字节表示，特殊的第一位为0，单独一个字节为一个ASCII字符。

很显然这种编码方式完全兼容ASCII码，而这种编码方式唯一受益的就是使用ASCII码的国家(→_→)，他们几乎不用修改任何内容就能让他们的网站被各国浏览，欧洲国家还是得用两个字节来编码，汉字一般需要三个字节(孤僻汉字可能要用到四个字节)。所以对于一些欧洲国家，他们的网站如果没有国际化的要求仍可以使用ISO-8895编码方案；对于中国网站，不需要国际化也仍可以使用GBK。

## 7.2、[UTF-16](https://en.wikipedia.org/wiki/UTF-16)

### 7.2.1、UTF-16代理区

在BMP中有一个区域：U+D800~U+DFFF，被称为UTF-16代理区(UTF-16 Surrogates)。

因为Unicode字符集的编码值范围为0-0x10FFFF(17个平面)，而大于0xFFFF的平面区码点值无法用2个字节来表示，所以Unicode标准规定：BMP中U+D800~U+DFFF范围内的值不对应于任何字符，为代理区。

这个代理区在下面讲述UTF-16的编码方式的时候会有很大的作用。

> 前面讲Unicode平面的时候给了一张BMP分布图，可以参照这张图来理解代理区的概念。

### 7.2.2、UTF-16的编码方式

**1.** 对于BMP平面的字符(也就是小于等于0xFFFF的码点，因为UTF-16代理区不对应任何字符，所以这里不包含代理区)：
    编码方式与UCS-2完全相同，直接使用字符码点值编码。
**2.** 对于其他平面的字符(也就是大于0xFFFF的码点)：

   * 将码点值减去0x10000，得到的值的范围为0~0xFFFFF，也就是可以用20位表示的值。(因为Unicode分为17个平面，最大码点值为0x10FFFF，减去一个0x10000，最大值为0xFFFFF)，这个20位的值写成二进制：HHHHHHHHHHLLLLLLLLLL。
   * 将相减得到的20位的高10位加上0xD800得到一个16位的码元，这个由高10位得到的码元称为**高位代理(high surrogate)**，它的范围为：0xD800~0xDBFF（0xD800加上一个10比特的数）。由于高位代理比低位代理的值要小，所以为了避免混淆使用，Unicode标准现在称高位代理为**前导代理(lead surrogates)**。
   * 将相减得到的20位的低10位加上0xDC00得到另一个16位的码元，这个由低10位得到的码元称为**低位代理(low surrogate)**，它的范围为：0xDC00~0xDFFF（0xDC00加上一个10比特的数）。由于低位代理比高位代理的值要大，所以为了避免混淆使用，Unicode标准现在称低位代理为**后尾代理(trail surrogates)**。
   * 由其他平面的码点值最终的到的UTF-16编码的二进制表示为：`110110HHHHHHHHHH` `110111LLLLLLLLLL`。

> 由UTF-16的编码方式可以看出UTF-16编码完全兼容UCS-2编码的，可以说UCS-2是UTF-16编码方式的一个子集。这也是设计UTF-16的目的，因为Unicode刚推出来时，大部分软件都是使用UCS-2，比如微软的Windows操作系统专门为Unicode重新设计了一份API，刚开始就是使用UCS-2；Java的Character和String等字符处理类最初都是使用UCS-2定长编码。
>
> 而且前导代理(0xD800~0xDBFF)、后尾代理(0xDC00~0xDFFF)、BMP有效码元(0x0000~0xD7FF和0xE000~0xFFFF)，三者互不重叠，意味着任意位置的16位码元都可以确定边界，也就是说UTF-16和UTF-8一样是自同步的，可以通过检查码元可以判定给定字符的下一个字符的起始码元。

## 7.3 UTF-32

相对于UTF-8和UTF-16而言，UTF-32使用场合的就比较少了，因为BMP平面的字符基本能满足日常用字的需要，所以使用UTF-32存储无疑是巨大的空间浪费。在存储方式上UTF-32与UCS-4一样使用4个字节存储数据，所以UTF-32和UCS-4一样也是定长编码，区别在于UTF-32的前导比特(前面的11位)必须为0，只能表示$2^{21}$个Unicode字符，在形式上UTF-32是UCS-4的子集。

###  7.3.1、小端序与大端序

由于UTF-16是以16位作为码元，也就是两个字节作为一个单位，这就涉及到一个问题：高位字节在前，还是低位字节在前？

先来了解一下两个概念：小端存储和大端存储。

* 由于CPU对数据进行运算是以字节为单位，将数据从内存读取到CPU的算术逻辑单元(ALU)中：**先读低位字节，后读高位字节**。所以数据按照**低字节存放在低地址，高字节存放在高地址**的原则存储在内存中的，CPU从低地址向高地址读取数据的时候就会先读取到低位字节的数据。这就是所谓的**“小端存储”**(little-endian，缩写为LE)，为了方便记忆也可以叫做“低尾端(低字节在后的意思)”。

  ![小端存储](http://tva1.sinaimg.cn/large/bda5cd74ly1g0wx9cyf5aj20dg06dt8o.jpg)

* 显然小端存储不符合人类的直观读取顺序，所以对于文件存储以及网络传输这种与CPU接触较少的情况一般都是使用**大端存储(big-endian，缩写为BE)**：**高字节存放在低地址，低字节存放在高地址**。为了方便记忆也可以叫做“高尾端(高位字节在后的意思)”。

  ![大端序](http://tva1.sinaimg.cn/large/bda5cd74ly1g0wx9uex86j20dg06dgll.jpg)

### 7.3.2、BOM字节序标记

UTF-16由两个字节组成，在存储或运算的时候有大端序、小端序的分别。但是计算机如何识别UTF-16的文本是大端序还是小端序呢。于是在UTF-16文本文件前面加两个字节的标记来标识该文件是小端序还是大端序，这个标记叫BOM(Byte Order Mark，字节序标记)。这两个字节的标记为`FE FF`则表示是大端序(高位在后)；如果标记为`FF FE`则表示是小端序(低位在后)。

相应地，UTF-32也有四个字节的BOM标记，四个字节值为`00 00 FE FF`表示是大端序，四个字节值为`FF FE 00 00`表示小端序。

> UTF-8虽然以单字节作为码元，但也可以有BOM标记`EF BB BF`，UTF-8的BOM不是用来标记字节序的，而仅仅用作标识UTF-8文件。

## 7.4、验证Unicode编码

Unicode中第一个汉字为“一”，码点值为U+4E00。

1. 我们用记事本写下“一”。

![记事本](http://tva1.sinaimg.cn/large/bda5cd74ly1g0wxbaclaoj208d0573yp.jpg)

2. 选择保存格式

   ![保存格式](http://tva1.sinaimg.cn/large/bda5cd74ly1g0wxbi45p8j20gs05sjrl.jpg)

3. 保存为UTF-8后，以二进制方式打开

   ![UTF-8](http://tva1.sinaimg.cn/large/bda5cd74ly1g0wxbqu2skj208l02kjr7.jpg)

   `EF BB BF`为UTF-8文件的BOM标记，`E4 B8 80`的二进制表示方式为`11100100 10111000 10000000`。去掉UTF-8的标识信息就剩下：`00100 111000 000000`，这个二进制对应十六进制为4E00。

4. 保存为Unicode，也就是UTF-16LE小端存储方式，然后以二进制方式打开

   ![UTF-16LE](http://tva1.sinaimg.cn/large/bda5cd74ly1g0wxbzua9ej208402mdfp.jpg)

   `FF FE`标识该文件是UTF-16小端存储。`00 4E`对应4E00

5. 保存为Unicode big endian，也就是UTF-16BE大端存储方式，然后以二进制方式打开

   ![UTF-16BE](http://tva1.sinaimg.cn/large/bda5cd74ly1g0wxcd5tzlj207i0293ya.jpg)

   `FE FF`标识该文件是UTF-16大端存储。`4E00`就是“一”的码点值。

从这里也可以看出，中文以UTF-16方式存储所占空间比UTF-8所占空间更小。

# 参考文献

维基百科：相关链接在文章中已经给出

各种编码详解：https://my.oschina.net/liting/blog/470021

