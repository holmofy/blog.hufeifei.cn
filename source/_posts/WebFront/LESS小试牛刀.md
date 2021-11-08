---
title: LESS小试牛刀
date: 2017-09-12
categories: 前端
---

##为什么选择less

CSS代码开发与维护都比较困难，特别是CSS中的各种尺寸颜色，看多了绝对想吐。所以就有了便于开发以及维护管理的CSS预处理语言，可以由它们编译生成CSS。

作为一个搞后端的Javer，我所了解的CSS预处理语言大致有三种[LESS](https://en.wikipedia.org/wiki/Less_(stylesheet_language)), [SASS](https://en.wikipedia.org/wiki/Sass_(stylesheet_language)), [Stylus](https://en.wikipedia.org/wiki/Stylus_(stylesheet_language))。这几种预处理语言都提供了变量(variables)，混合书写(mixins)，函数(functions)，运算(operations)等功能，这些功能大大降低了CSS的编写难度。在这几种语言中[SASS](sass-lang.com)基于Ruby，[LESS](http://lesscss.org)、[Stylus](http://stylus-lang.com)基于Node.js。Ruby接触的不多，为了节约学习成本果断舍弃SASS。Stylus属于后起之秀，目前在[github](https://github.com/stylus)上活跃度不如[LESS](https://github.com/less)和[SASS](https://github.com/sass)。另外，LESS语法完全兼容CSS，学习周期更短(SASS 3后也改版成了兼容CSS的SCSS语法)，并且Bootstrap框架也使用LESS语言。权衡利弊，最终我还是选择了LESS。

> Less官网列出了使用了less的框架：http://lesscss.org/usage/#frameworks-using-less

##安装LESS

less是基于node开发的，所以安装前确保已经有[Node.js](https://nodejs.org/)的开发环境。

使用node.js的包管理器全局安装：

```shell
$ npm install -g less
```

> 全局安装会装在Node全局目录，如果全局目录已设环境变量，则可以在任何目录下使用

安装完成后即可使用Less编译器将less源文件编译成css：

```shell
$ lessc styles.less styles.css
```

lessc命令内部提供了`-x`选项用于生成压缩的CSS。在V3版本后lessc提供了将压缩功能抽离成了一个[`clean-css`](https://github.com/less/less-plugin-clean-css)插件，安装`clean-css`命令如下：

```shell
$ npm install -g less-plugin-clean-css
```

插件安装后使用以下命令，lessc会自动调用插件：

```shell
$ lessc --clean-css styles.less styles.min.css
```

> 除了命令行的编译方式，less官方还提供了
>
> * 在线编译环境：http://less2css.org/，
> * 第三方GUI编译环境：http://lesscss.org/usage/#guis-for-less

##使用less.js进行前端开发

> less和less.js适用于开发阶段的调试，实际发布为了能得到更好的性能应该使用使用编译后的css

将less样式文件链入对应的html文件中：

```html
<link rel="stylesheet/less" type="text/css" href="styles.less" />
```

下载[less.js](https://github.com/less/less.js/archive/master.zip)，并在html文件中引入：

```html
<script src="less.js" type="text/javascript"></script>
```

> 注意：1. 确保样式文件在脚本文件前；2. 每个`.less`都是单独编译的，所以里面的变量或混合选项等都不能共享；3. 由于浏览器的同源策略所以加载外部资源要求开启跨域请求。

可选项：可以在加载less.js文件之前为less进行相应的配置

```html
<!-- 需要在 less.js 脚本之前设置 -->
<script>
  less = {
    env: "development",
    async: false,
    fileAsync: false,
    poll: 1000,
    functions: {},
    dumpLineNumbers: "comments",
    relativeUrls: false,
    rootpath: ":/a.com/"
  };
</script>
<script src="less.js"></script>
```

> [点击这里](http://lesscss.org/usage/#using-less-in-the-browser-setting-options)有该配置的相关解释

##变量：variables

## 变量定义

对于CSS中经常出现的共用属性值，我们可以使用变量来统一管理：

```css
a, .link { color: #428bca; }
.widget { color: #fff; background: #428bca; }
```

使用LESS变量统一管理：

```less
// 定义变量
@link-color:        #428bca; // sea blue
@link-color-hover:  darken(@link-color, 10%); // 使用内建函数创建相关颜色

// 使用变量
a, .link { color: @link-color; }
a:hover { color: @link-color-hover; }
.widget { color: #fff; background: @link-color; }
```

> LESS中的变量和编程语言中的变量不同，我们并不能修改变量的值，只能在相应的地方引用该变量。所以更准确的定义应该叫“常量”。

## 变量插值语句

上面的例子是将变量用作“属性值”，事实上变量也能用在其他地方，比如选择器、属性名、URL或@import指令。

```less
// 定义变量
@my-selector: banner;

// 变量用作选择器
.@{my-selector} {  font-weight: bold; line-height: 40px; margin: 0 auto; }

// 定义变量
@images: "../img";
// 在URL中使用变量
body { color: #444; background: url("@{images}/white-sand.png"); }

// 定义变量
@themes: "../../src/themes";

// 在@import指令中使用变量
@import "@{themes}/tidal-wave.less";

// 定义变量
@property: color;

// 在属性名中使用
.widget {
  @{property}: #0ee;
  background-@{property}: #999;
}
```

> 变量用作选择器和属性名时必须要以`@{ }`的方式引用变量，否则编译不通过。在url和@import指令中使用也需要`@{ }`，否则会被当成字符串的一部分。

## 变量用作变量名

这句话可能有点绕，我觉的看例子最直接：

```less
@fnord:  "I am fnord.";
@var:    "fnord";
content: @@var;  // @@var=@fnord=I am fnord.
```

最终会被编译成：

```less
content: "I am fnord.";
```

## 变量的惰性加载

可以在变量的定义之前使用变量

```less
.lazy-eval {
  width: @var; // 在变量定义之前使用
}

@var: @a;
@a: 9%;
```

最终编译成：

```less
.lazy-eval-scope {
  width: 9%;
}
```

你甚至可以这样：

```less
.lazy-eval-scope {
  width: @var;
  @a: 9%;
}

@var: @a;
@a: 100%;
```

同一作用域内，后定义的变量会覆盖先定义的变量，所以：

```less
@var: @a;
.lazy-eval-scope {
  width: @var;
}
@a: 100%;
@a: 99%;
```

会被编译成：

```less
.lazy-eval-scope {
  width: 99%;
}
```

不同作用域则采用就近原则：

```less
@var: @a;
.lazy-eval-scope {
  width: @var;
  @a: 9%;
}
@a: 100%;
@a: 99%;
```

会被编译成：

```less
.lazy-eval-scope {
  width: 9%;
}
```

## 默认变量的覆盖

使用第三方框架时，我们可能需要修改里面的一些变量属性值，根据上面变量惰性加载的原理，我们可以这样使用：

```less
// 第三方框架中的变量
@base-color: green;
@dark-color: darken(@base-color, 10%);

// 使用第三方库
@import "library.less";
@base-color: red;
// 根据变量惰性加载的原理，red会覆盖框架中定义的变量
// 所以@dark-color变量也会被修改成暗红色drak-red
// 这样可以保证我们使用第三方框架时不用修改框架中任何代码
```

##继承：extend

extend是less中的一个伪类选择器，用于合并共用CSS样式。

> less中的extend被设计成了伪类选择器，相对于sass，这也是less被诟病的地方。

```less
nav ul {
  // 这里的&符号表示选择器本身
  &:extend(.inline);
  background: blue;
}
.inline {
  color: red;
}
```

会被编译成：

```less
nav ul {
  background: blue;
}
.inline,
nav ul {
  color: red;
}
```

> 当被继承的类选择器(这里是`.inline`)不存在，或者选择器中没有任何CSS属性，则该继承类在编译时会被删除。

## 继承的语法规则

```less
////////////////
// 最基本的写法
.a:extend(.b) {}
.some-class:extend(.bucket tr) {}

// 和上面的写法等价
.a {
  &:extend(.b);
}

////////////////
// all 关键字表示继承所有的.d类选择器
.c:extend(.d all) {
  // 比如.d 或 .x.d 或 .d.x
  // 前面的写法只会继承.d选择器
}
// 比如：
.a.class,
.class.a,
.class > .a {
  color: blue;
}
.testa:extend(.class) {} // 未找到匹配的选择器
.testb:extend(.class all) {} // 匹配所有包括.class的选择器

////////////////
// 多继承
.e:extend(.f) {}
.e:extend(.g) {}
// 继承多个类选择器可以用逗号隔开
// 注意，有没有逗号区别很大
.e:extend(.f, .g) {}
// 这种多继承语法也是允许的
.e:extend(.f):extend(.g) {}
```

>extend只能放在选择器的末尾，`a:hover:extend(.link).nth-child(odd)`这种写法是不被允许的。

##样式混合：mixins

样式混合就是把已有的样式插入到选择器中

```less
.a, #b {
  color: red;
}
.mixin-class {
  .a();
}
.mixin-id {
  #b();
}
```

将会编译成：

```less
.a, #b {
  color: red;
}
.mixin-class {
  color: red;
}
.mixin-id {
  color: red;
}
```

> 注意样式混合与样式继承的关系，不要混淆。

无输出的样式混合：

```less
.my-mixin {
  color: black;
}
// 这一部分不会被输出
.my-other-mixin() {
  background: white;
}
.class {
  .my-mixin;
  .my-other-mixin;
}
// 混合样式后的括号可加可不加
```

会被编译成：

```less
.my-mixin {
  color: black;
}
.class {
  color: black;
  background: white;
}
```

## 带参数的mixins

```less
// 定义mixins
.border-radius(@radius) {
  -webkit-border-radius: @radius;
     -moz-border-radius: @radius;
          border-radius: @radius;
}

/////
// 使用mixins
#header {
  .border-radius(4px);
}
.button {
  .border-radius(6px);
}
```

## mixins默认参数

```less
.border-radius(@radius: 5px) {
  -webkit-border-radius: @radius;
     -moz-border-radius: @radius;
          border-radius: @radius;
}

///////
// 直接使用默认参数
#header {
  .border-radius;
}
```

## 多参mixins

```less
// 多个变量之间也使用","隔开
.caret-down(@width: 10px, @color: #ccc){
    border-width: @width;
    border-color: @color transparent transparent transparent;
    border-style: solid dashed dashed dashed;
}

// 多个变量之间也可以使用";"隔开
.caret-down(@width: 10px, @color: #ccc){
    border-width: @width;
    border-color: @color transparent transparent transparent;
    border-style: solid dashed dashed dashed;
}
```

> 推荐使用";"分隔参数，因为","还可能是多个css属性值的分隔符

## @arguments变量

@arguments变量类似于JS中的arguments，用来表示mixins的所有输入参数。

```less
// 定义mixins
.box-shadow(@x: 0; @y: 0; @blur: 1px; @color: #000) {
  -webkit-box-shadow: @arguments;
     -moz-box-shadow: @arguments;
          box-shadow: @arguments;
}

/////
// 使用
.big-block {
  .box-shadow(2px; 5px);
}
```

被编译成：

```less
.big-block {
  -webkit-box-shadow: 2px 5px 1px #000;
     -moz-box-shadow: 2px 5px 1px #000;
          box-shadow: 2px 5px 1px #000;
}
```

## 可变参数

像编程语言一样可以使用`...`表示可变参数：

```less
.mixin(...) {        /* 匹配 0-N 个参数 */  }
.mixin() {           /* 匹配 0 个参数 */    }
.mixin(@a: 1) {      /* 匹配 0-1 个参数 */  }
.mixin(@a: 1; ...) { /* 匹配 0-N 个参数 */  }
.mixin(@a; ...) {    /* 匹配 1-N 参数 */    }
```

示例：

```less
.mixin(@a; @rest...) {
    content: @rest;
}
```



> LESS文档地址：http://lesscss.org/