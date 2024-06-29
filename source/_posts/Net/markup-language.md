---
title: 标记语言的历史
date: 2024-03-30
tags: 
 - 标记语言
 - Markdown
 - Asciidoc
 - Tex
categories: 网络
---

说起编程语言，大家都知道，C、C++、Java、Python、C#，对[编程语言][Programming_language]了解的深的，可能还能说出1957年就被发明的[Fortran][Fortran]，Fortran语言仍是美国学术界使用的编程语言，至今仍在[编程语言排行榜][tiobe-index]霸榜前十。

但说起[标记语言][Markup_language]，大家可能就知道一个HTML，在互联网上文章写得多的可能还知道Markdown，我这个网站的文章都是用Markdown写的。由于Markdown不支持内嵌外部代码，这几天找到了Spring也在用的Asciidoc准备作为开源库的文档标记语言。正好了解到标记语言的发展历史，就写篇文章聊聊。

## 数字出版业的GenCode

可能说出来有人不信，标记语言的历史远比大多数编程语言早————第一门标记语言比C语言还早出生几年。当时的计算机还是稀有物品，出版业就是使用计算机的一大客户。出版业使用一个[TYPSET][TYPSET]的软件用来做排版，TYPSET编辑器和RUNOFF排版软件一起使用，这可能是历史上最早的文档编辑器了。

一个叫[威廉][William]的针对RUNOFF程序，为出版行业制定了最早的文本标记语言的标准GenCode，这个人后来也成了[ISO标准委员会](ISO)的第一任主席。

## 通用标记语言: XML和HTML的前身

[ISO标准委员会](ISO)，学工科的应该都知道，工业界的大多数国际标准都是这个[国际组织][ISO_org]制定的。咱计算机领域的OSI七层网络协议就是这个组织制定的。

1986年，ISO组织就基于IBM的[通用标记语言GML][GML]制定了第一个国际化标记语言的标准[SGML][SGML]。SGML就是众所周知的XML和HTML的前身。

这个[GML][GML]是IBM研究员[查尔斯][Charles]在1969年发明的，SGML基于GML和GenCode，也是[查尔斯][Charles]进入[ISO委员会](ISO)制定的第一个标准。[查尔斯][Charles]也因此被称为“标记语言之父”、“HTML之祖”。

## 学术界的标记语言: Tex

大家都知道C语言是[贝尔实验室][Bell-Labs]的[丹尼斯·里奇][Dennis_Ritchie]发明的，他和[肯·汤普森][Ken_Thompson]在参与[Multics][Multics]操作系统研发失败后回到了贝尔实验室。[肯·汤普森][Ken_Thompson]为了用一台废旧的PDP-7小型机玩游戏，开发了[Unix操作系统][Unix]和[B语言][B]。[丹尼斯·里奇][Dennis_Ritchie]为了改进系统，又开发了[C语言][C]重写了[Unix系统][Unix]。

当时的[贝尔实验室][Bell-Labs]大拿云集，很多人被Unix操作系统吸引，在它上面开发各种工具，[Troff][Troff]和[Nroff][Nroff]就是Unix上的排版软件，你可以把这个软件可以类比为Windows上的Word。

Unix早期还没有受到贝尔公司的重视，没有商业化，除了在贝尔实验室流行开来，还流传到了大学里。[加州大学][UC]就是其中之一，[加州大学伯克利分校][UC-Berkeley]在Unix基础上进行了二次开发，随着Unix被各大商业公司拿去搞钱弄成了闭源，伯克利重写了Unix操作系统，现在流行的Unix开源版本[Open BSD][BSD]就是这个学校开发的。

当时大学里很多人要写论文，论文里有很多数学公式，为了解决论文的排版问题，1978年~1989年间在斯坦福大学当教授的[唐纳德·克努斯][Donald_Knuth]开发了[Tex][TeX]，斯坦福研究所的[莱斯利·兰波特][Leslie_Lamport]认为Tex可以通用起来，就自己开发了[Latex][Latex]并在1986年发布问世。[Tex排版系统][Tex-format]现在几乎成了学术界标准的排版软件。后来有人把[Tex][TeX]移植到Unix操作系统上取代了[Troff][Troff]和[Nroff][Nroff]。

![Tex排版软件](https://pic1.zhimg.com/v2-d56d408d8ff9c02480db5795bb7a20f8_r.jpg)

Tex也有几个Web的移植版本，[Letex.js][latex.js]、[MathJax][mathjax]、[KaTex.js][katex]。我这个网站上的公式渲染就是用了MathJax。

最近有一款新的标记语言非常火，[typst](https://github.com/typst/typst)号称要取代Latex。截止2024年5月，这个项目已经有2万多star了，势头很猛。

![typst star history](https://api.star-history.com/svg?repos=typst/typst&type=Date)

## 互联网让HTML风靡起来

1980年，在欧洲核子研究中心任职的物理学家[伯纳斯-李][Berners-Lee]为了方便研究中心人员共享文档，发明了[HTML][HTML]，并找了个本科生[尼古拉·佩洛][Nicola_Pellow]写了第一个网络浏览器[Line Mode Browser][Line_Mode_Browser]，这个浏览器当时还是在终端运行。后来的1990年[伯纳斯-李][Berners-Lee]开发了第一个Web服务器[CERNhttpd][cern-httpd]。现在还能在欧洲核子研究中心看到世界上第一个网站：[http://info.cern.ch/](https://info.cern.ch/) 和 第一个浏览器[line-mode](Line_Mode)的历史。能翻墙的话也可以到维基百科上看看[万维网的历史][History_of_the_World_Wide_Web]。

![第一个网络浏览器](https://ts1.cn.mm.bing.net/th/id/R-C.02c1e2425a433d2176da44a046b0fb7e?rik=ELDtCH4JacUGdw)

后来就是图形界面浏览器，以及众所周知的网景与微软的IE浏览器大战了。

1991年Sun公司开发的Java语言就是乘着这一波互联网东风风靡起来的。1996年IE支持Java Applets本来的想法和今天的微信小程序一样，结果Java在浏览器上没掀起什么波浪，在服务端却大放异彩。1997年IE支持了Ajax，互联网迈入Web2.0时代，[Dynamic HTML][Dynamic_HTML]开始流行。客户端脚本语言Javascript蹭了Java的热度，成了现今浏览器的标准；服务端脚本PHP, Python, Java的JSP 和 后来C#的ASP.NET陆陆续续出现。美国的门户网站雅虎、搜索引擎谷歌，中国的三大门户网易、搜狐、新浪，搜索引擎百度就是这个时候崛起的，腾讯、阿里也是这个时候初创还未成气候。

2000年互联网泡沫破裂，大量互联网公司因为融不到资，最后资金链断裂倒闭。这场风暴后活下来的公司，很多现在都成了世界五百强。很多以前的网页现在只能通过 https://archive.org/ 互联网档案馆这个网站看到。

现在的互联网网站比以前绚烂多彩。浏览器从IE一家独大又变回了百花齐放；HTML已经进化到HTML5标准；CSS已经进化到CSS3，CSS4标准也陆续出台；JS也树立了ECMAScript标准，每年都会有新的规范出台；Web上XML也有了用于公式渲染的MathML 和 矢量图形渲染的SVG等一系列子集。连Office办公套件也在2007版之后使用[OpenXML][OpenXML]规范了，现在看到的docx、xlsx、pptx都是个xml文件的压缩包。

1980年代的C++、1990年代的Java也在新时代的浪潮下不断推出新版本，但是由于历史包袱太重，2010年后Go、Rust、Swift、Zig等新起之秀也相继出现。继移动互联网后，新一代人工智能浪潮又来了，估计大多数程序员和我一样觉得互联网的变化太快了。

## 轻量标记语言

随着互联网崛起的HTML，有个缺点就是太重了，这是所有XML的通病。早期的Ajax使用XML传输数据，微软的IE提供的Ajax接口就叫做`XMLHttpRequest`，就是因为XML太重了，2001年才基于Javascript开发了[JSON][JSON]（JavaScript Object Notation），也叫做JSML(JavaScript Markup Language)，现在几乎成了互联网传输数据的标准。

在渲染方面HTML也太重，特别是对于非计算机专业的人士来说，HTML太难懂了，所以发展出了**轻量级标记语言**。

最早的轻量级标记语言，主要是用在论坛发布广告栏，管理员不用懂复杂的HTML嵌套语法，在后台直接基于[BBCode][BBCode]或[Textile][Textile]写公告。

![BBCode](https://fredcrash.com/bbcode/images/exemplebbcode.png)

还有一个应用场景是渲染电子邮件和软件的文档。约1992年开发的[Setext][Setext]就是用来写邮件和评论帖子的轻量标记语言，2002年开发的[ATX][ATX]是用来写博客的，[reStructuredText][ReStructuredText]就是Python用于生成文档的标记语言

现在依然活跃的[Textile][Textile]是2002年开发的，一些博客或CMS网站上还在用。这种轻量级标记语言语法简单，是个人分分钟就能学会。基于这个轻量级标记语言再渲染成HTML，可以节省很多排版工作。

现在非常流行的Markdown，从名字可以看出和Markup标记语言对应，它是John Gruber受[Setext][Setext]、[Textile][Textile]和[reStructuredText][ReStructuredText]影响在2004年开发的。Markdown被[Github(**G**itHub **F**lavored **M**arkdown)][GFM]、Stack Exchange、SourceForge等网站采用，并加了很多额外的[扩展语法][extended-markdown]因而逐渐分裂，已经有几十个版本的[Markdown解析器实现][Markdown-Implementations]。所以2012年一群人发起了[CommonMark][CommonMark]计划对Markdown进行了标准化，在[commonmark.org](https://commonmark.org/help/)网站可以看到相应的语法规范。

Markdown基本语法还是很简单的，CommonMark标准2024年已经发展到[`0.31.2`](https://spec.commonmark.org/0.31.2/)版了，可以看到CommonMark是兼容ATX和Setext语法的。实现了CommonMark语法规范的解析器实现可以在它的[wiki][CommonMark-Implementations]上看到。

轻量级标记语言已经有很多种了，截止2024年Wikipedia上已经列出了二十多种轻量级标记语言：

| Language                     | HTML export tool | HTML import tool | Tables  | Link titles | class attribute | id attribute | Release date     |
|------------------------------|------------------|------------------|---------|-------------|-----------------|--------------|------------------|
| setext                       | Yes              | Yes              | No      | Yes         | No              | No           | 1992         |
| POD                          | Yes              | ?                | No      | Yes         | ?               | ?            | 1994             |
| BBCode                       | No               | No               | Yes     | No          | No              | No           | 1998             |
| txt2tags                     | Yes              | Yes              | Yes     | Yes         | Yes/No          | Yes/No       | 2001-07-26   |
| MediaWiki                    | Yes              | Yes              | Yes     | Yes         | Yes             | Yes          | 2002         |
| PmWiki                       | Yes              | Yes              | Yes     | Yes         | Yes             | Yes          | 2002-01          |
| reStructuredText             | Yes              | Yes              | Yes     | Yes         | Yes             | auto         | 2002-04-02   |
| AsciiDoc                     | Yes              | Yes              | Yes     | Yes         | Yes             | Yes          | 2002-11-25    |
| Textile                      | Yes              | No               | Yes     | Yes         | Yes             | Yes          | 2002-12-26   |
| Jira Formatting Notation     | Yes              | No               | Yes     | Yes         | No              | No           | 2002+         |
| Org-mode                     | Yes              | Yes              | Yes     | Yes         | Yes             | Yes          | 2003         |
| Texy                         | Yes              | Yes              | Yes     | Yes         | Yes             | Yes          | 2004         |
| **Markdown**                 | Yes              | Yes              | No      | Yes         | Yes/No          | Yes/No       | 2004-03-19 |
| TiddlyWiki                   | Yes              | No               | Yes     | Yes         | Yes             | No           | 2004-09      |
| Creole                       | No               | No               | Yes     | No          | No              | No           | 2007-07-04    |
| MultiMarkdown                | Yes              | No               | Yes     | Yes         | No              | No           | 2009-07-13       |
| **GitHub Flavored Markdown** | Yes              | No               | Yes     | Yes         | No              | No           | 2011-04-28+      |
| **Markdown Extra**           | Yes              | Yes              | Yes     | Yes         | Yes             | Yes          | 2013-04-11    |
| Slack                        | No               | No               | No      | Yes         | No              | No           | 2013+    |
| WhatsApp                     | No               | No               | No      | No          | No              | No           | 2016-03-16   |
| Gemtext                      | Yes              | ?                | No      | Yes         | No              | No           | 2020             |
| Djot                         | Yes              | Yes              | Yes     | Yes         | Yes             | Yes          | 2022-07-30    |

## 文档的相互转换

大家都知道Markdown可以导出HTML和pdf，实际上有一个[pandoc][pandoc]项目，可以在市面上大多数文档软件格式之间进行转换。

项目是用haskell写的：https://github.com/jgm/pandoc 。它甚至支持word、ppt的格式转换。

![pandoc star history](https://api.star-history.com/svg?repos=jgm/pandoc&type=Date)

目前来看，最流行的markdown不仅用在写技术文档，还在知乎等社交论坛网站有应用。不过目前对我来说最大的痛点是无法include外部文件，这个功能2014年在[CommonMark中][talk.commonmark]就有过讨论了，但是一直都没有标准化的实现。所以在技术文档方面asciidoc会更强大一些，最有名的就是Spring，它的所有文档都是基于asciidoc。github也支持渲染asciidoc文档。



[Markup_language]: https://en.wikipedia.org/wiki/Markup_language
[Programming_language]: https://en.wikipedia.org/wiki/Programming_language
[Fortran]: https://fortran-lang.org/en/index.html
[tiobe-index]: https://www.tiobe.com/tiobe-index
[TYPSET]: https://en.wikipedia.org/wiki/TYPSET_and_RUNOFF
[William]: https://en.wikipedia.org/wiki/William_W._Tunnicliffe
[ISO]: https://en.wikipedia.org/wiki/International_Organization_for_Standardization
[ISO_org]: https://www.iso.org/about
[GML]: https://en.wikipedia.org/wiki/IBM_Generalized_Markup_Language
[SGML]: https://en.wikipedia.org/wiki/Standard_Generalized_Markup_Language
[Charles]: https://en.wikipedia.org/wiki/Charles_Goldfarb
[Bell-Labs]: https://baike.baidu.com/item/贝尔实验室/686816
[Dennis_Ritchie]: https://en.wikipedia.org/wiki/Dennis_Ritchie
[Ken_Thompson]: https://en.wikipedia.org/wiki/Ken_Thompson
[Multics]: https://en.wikipedia.org/wiki/Multics
[Unix]: https://en.wikipedia.org/wiki/Unix
[B]: https://en.wikipedia.org/wiki/B_(programming_language)
[C]: https://en.wikipedia.org/wiki/C_(programming_language)
[Troff]: https://en.wikipedia.org/wiki/Troff
[Nroff]: https://en.wikipedia.org/wiki/Nroff
[UC]: https://baike.baidu.com/item/加利福尼亚大学/3135904
[UC-Berkeley]: https://baike.baidu.com/item/加利福尼亚大学伯克利分校/854061
[BSD]: https://baike.baidu.com/item/伯克利软件套件/63038897
[Donald_Knuth]: https://en.wikipedia.org/wiki/Donald_Knuth
[TeX]: https://en.wikipedia.org/wiki/TeX
[Leslie_Lamport]: https://en.wikipedia.org/wiki/Leslie_Lamport
[Tex-format]: https://baike.baidu.com/item/TeX/3794463
[Latex]: https://en.wikipedia.org/wiki/LaTeX
[latex.js]: https://latex.js.org/
[mathjax]: https://www.mathjax.org/
[katex]: https://katex.org/
[Berners-Lee]: https://en.wikipedia.org/wiki/Tim_Berners-Lee
[Nicola_Pellow]: https://en.wikipedia.org/wiki/Nicola_Pellow
[HTML]: https://en.wikipedia.org/wiki/HTML
[cern-httpd]: https://baike.baidu.com/item/cern%20httpd/16747491
[Line_Mode_Browser]: https://en.wikipedia.org/wiki/Line_Mode_Browser
[Line_Mode]: https://line-mode.cern.ch/
[History_of_the_World_Wide_Web]: https://en.wikipedia.org/wiki/History_of_the_World_Wide_Web
[Dynamic_HTML]: https://en.wikipedia.org/wiki/Dynamic_HTML
[JSON]: https://en.wikipedia.org/wiki/JSON
[OpenXML]: https://support.microsoft.com/en-us/office/open-xml-formats-and-file-name-extensions-5200d93c-3449-4380-8e11-31ef14555b18
[Lightweight_markup_language]: https://en.wikipedia.org/wiki/Lightweight_markup_language
[BBCode]: https://baike.baidu.com/item/BBCode/6814117
[Setext]: https://en.wikipedia.org/wiki/Setext
[ATX]: http://www.aaronsw.com/2002/atx/intro
[Textile]: https://textile-lang.com/
[ReStructuredText]: https://www.sphinx-doc.org/en/master/usage/restructuredtext/basics.html
[GFM]: https://github.github.com/gfm/
[extended-markdown]: https://www.markdownguide.org/extended-syntax/
[Markdown-Implementations]: https://github.com/markdown/markdown.github.com/wiki/Implementations
[CommonMark]: https://commonmark.org/
[CommonMark-Implementations]: https://github.com/commonmark/commonmark-spec/wiki/List-of-CommonMark-Implementations
[pandoc]: https://pandoc.org/
[talk.commonmark]: https://talk.commonmark.org/t/transclusion-or-including-sub-documents-for-reuse/270/8
