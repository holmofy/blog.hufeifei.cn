---
title: 将Sublime打造成超强前端IDE
date: 2017-08-12
categories: 前端
---

前几天把Sublime更新了一下，插件什么的重新装，但是突然都忘了以前装过什么插件了，把内容记录下来以备将来再查。

sublime下载地址：https://www.sublimetext.com/3

sublime文档地址：

* https://www.sublimetext.com/docs/3/
* http://docs.sublimetext.info

授权码（不激活也可以使用，推荐购买授权码）

```shell
—– BEGIN LICENSE —–
TwitterInc
200 User License
EA7E-890007
1D77F72E 390CDD93 4DCBA022 FAF60790
61AA12C0 A37081C5 D0316412 4584D136
94D7F7D4 95BC8C1C 527DA828 560BB037
D1EDDD8C AE7B379F 50C9D69D B35179EF
2FE898C4 8E4277A8 555CE714 E1FB0E43
D5D52613 C3D12E98 BC49967F 7652EED2
9D2D2E61 67610860 6D338B72 5CF95C69
E36B85CC 84991F19 7575D828 470A92AB
—— END LICENSE ——
```

#[Package Control](https://packagecontrol.io)

用于管理各种插件包的管理器，方便下载安装插件

```python
import urllib.request,os,hashlib; h = '6f4c264a24d933ce70df5dedcf1dcaee' + 'ebe013ee18cced0ef93d5f746d80ef60'; pf = 'Package Control.sublime-package'; ipp = sublime.installed_packages_path(); urllib.request.install_opener( urllib.request.build_opener( urllib.request.ProxyHandler()) ); by = urllib.request.urlopen( 'http://packagecontrol.io/' + pf.replace(' ', '%20')).read(); dh = hashlib.sha256(by).hexdigest(); print('Error validating download (got %s instead of %s), please try manual install' % (dh, h)) if dh != h else open(os.path.join( ipp, pf), 'wb' ).write(by)
```

# [Emmet](https://emmet.io/)

使用Emmet插件编写HTML,XML,CSS非常方便，简单的一个Tab键可以让你省很多事儿。而且Emmet插件支持各大主流编辑器和IDE。

![Emmet](http://img-blog.csdn.net/20170918131240723?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

这里有sublime text的emmet插件的相关配置：https://github.com/sergeche/emmet-sublime

# [SideBarEnhancements](https://github.com/titoBouzout/SideBarEnhancements)

丰富的侧边栏扩展：

![SideBarEnhancements](http://img-blog.csdn.net/20170918131313901?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

# [SublimeCodeIntel](https://github.com/SublimeCodeIntel/SublimeCodeIntel)

代码智能提示与自动补全，支持超过十种以上的编程语言。

![SublimeCodeIntel](http://img-blog.csdn.net/20170918131333986?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

# [SublimeLinter](http://www.sublimelinter.com/en/latest/)

这是一个代码纠错的框架，能够帮助你更快的清除语法错误。注意它仅仅只是一个框架，真正的纠错功能需要其他插件调用Linter程序，前端常用的linter插件有：

## 1、SublimeLinter-jshint

Github主页：https://github.com/SublimeLinter/SublimeLinter-jshint

使用之前需要安装JSLinter程序jshint，安装步骤如下：

* 安装Node.js
* 安装jshint：`npm install -g jshint`

## 2、SublimeLinter-csslint

Github主页：https://github.com/SublimeLinter/SublimeLinter-csslint

使用之前需要安装csslint，安装步骤如下：

* 安装Node.js
* 安装csslint：`npm install -g csslint`

## 3、SublimeLinter-contrib-htmlhint

Github主页：https://github.com/mmaday/SublimeLinter-contrib-htmlhint

使用之前需要安装htmlhint，安装步骤如下：

* 安装Node.js
* 安装htmlhint：`npm install -g htmlhint`

## 4、SublimeLinter-json

Github主页：https://github.com/SublimeLinter/SublimeLinter-json

SublimeLinter-json使用Sublime内建的JSON解析器所以不需要依赖其他软件。

# [DocBlock](https://github.com/spadgos/sublime-jsdocs)

用于编写文档注释的插件：

![DocBlock](https://camo.githubusercontent.com/087348d3e797f4ccc91528459b0473f6d34eadf3/687474703a2f2f73706164676f732e6769746875622e696f2f7375626c696d652d6a73646f63732f696d616765732f6c6f6e672d617267732e676966)

# [ConvertToUTF8](https://github.com/seanliang/ConvertToUTF8)

用于编码转换：

![ConvertToUTF8](https://camo.githubusercontent.com/50a5d366ba91aa9023fa55216f41f3cf5101e85d/68747470733a2f2f7365616e6c69616e672e6769746875622e696f2f646f6e6174652f436f6e76657274546f555446382e676966)

# [HTML-CSS-JS Prettify](https://github.com/victorporof/Sublime-HTMLPrettify)

该插件用于HTML,CSS,JS代码格式化，安装步骤如下：

1、安装Node.js(因为使用的JS-Beautify依赖于Node.js)

2、为插件配置Node路径



# [SublimeREPL](https://github.com/wuub/SublimeREPL)

该插件提供实时交互式命令行接口 *Read–Eval–Print Loop*

![SublimeREPL](https://camo.githubusercontent.com/6d88d10200e220c02e08cb7cd79757aa5adeeb5d/687474703a2f2f692e696d6775722e636f6d2f6d6d5951362e706e67)

# [LESS](https://github.com/danro/LESS-sublime) 和 [SASS](https://github.com/nathos/sass-textmate-bundle)

这两个插件分别提供LESS和SASS的语法高亮和自动补全功能。



# [TypeScript](https://github.com/Microsoft/TypeScript-Sublime-Plugin)

该插件由TypeScript的亲爸爸微软开发，提供了语法高亮，代码提示，代码格式化，语法纠错等各种功能。插件依赖Node.js，所以需要将Node设置到环境变量中。

![TypeScript](https://raw.githubusercontent.com/Microsoft/TypeScript-Sublime-Plugin/master/screenshots/errorlist.gif)