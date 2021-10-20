---
title: Babel学习笔记
date: 2018-03-15
categories: 前端
---

# babel有什么用

ES6标准从ES2015制定开始已经有几个年头了，虽然各大浏览器最新版都在极力地实现标准，但并不是所有人都会用最新版本的浏览器，特别是天朝像某狗某游这样的二次包装的浏览器，使用别人的内核而且版本更新又比较慢，导致新标准不能及时地在浏览器端使用。

Babel就是用来解决这个问题的：**将ES6的代码转换成ES5的代码，从而在现有的环境中运行，让我们能用下一代JS编写前端代码**。

```javascript
// 箭头函数是ES6标准中制定的，大部分老版本浏览器都不支持
arr.map(item => item+1)

// babel转码后
arr.map(function(item){
  return item+1;
})
```

> 想了解更多，来看看官网：https://babeljs.io/

# 插件(plugins)与预设(presets)

> 参考：https://babeljs.io/docs/plugins/

babel6之后将默认的转换功能做成了可插拔的插件，默认不携带任何转换功能，如果没有指定插件，babel会原样输出，不会进行转换。

**插件(plugins)只是单一的功能**，例如：

- es2015-arrow-functions
- es2015-classes
- es2015-for-of
- es2015-spread

![Babel官网列出ES2015的诸多特性](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbj4r2aszj20eq09j0t7.jpg)

> 这么多插件一个一个地安装，明显很不合理，所以大部分情况下我们都是用presets(预设)

**预设(presets)是一组插件**，例如：

* [babel-preset-es2015](https://babeljs.io/docs/plugins/preset-es2015/)(将ES2015语法编译成ES5)
* [babel-preset-2016](https://babeljs.io/docs/plugins/preset-es2016/)(将ES2016语法编译成ES2015)
* [babel-preset-2017](https://babeljs.io/docs/plugins/preset-es2017/)(将ES2017语法编译成ES2016)
* [babel-preset-latest](https://babeljs.io/docs/plugins/preset-latest/)(最新标准，将2015,2016,2017的语法编译成ES5，包含上面三个预设)
* [babel-preset-env](https://babeljs.io/docs/plugins/preset-env/)(Babel7中新推出的预设，可根据你指定的环境自动确定您需要的Babel插件)

> 最新版babel中babel-preset-latest被弃用，推荐使用babel-preset-env

* [babel-preset-react](https://babeljs.io/docs/plugins/preset-react/)(支持将React中的JSX转换成createElement的函数调用)

下面几个预设是TC39对ES的下一代提出的新功能的转换插件集合，这些功能还没有成为标准，有的功能甚至在将来会被移除，总之就是实验版预设(Experimental Presets)

- [preset-stage-3](https://babeljs.io/docs/plugins/preset-stage-3/)(五个插件)
- [preset-stage-2](https://babeljs.io/docs/plugins/preset-stage-2/)(三个插件以及3预设的功能)
- [preset-stage-1](https://babeljs.io/docs/plugins/preset-stage-1/)(两个插件以及2,3预设的功能)
- [preset-stage-0](https://babeljs.io/docs/plugins/preset-stage-0/)(包括1,2,3预设的功能)

> TC39对EmacScript的提案详见：https://github.com/tc39/proposals

# 在webpack构建系统中使用babel

用个babel有这么麻烦吗？光插件和预设就有这么多名堂。放心，我们不用管这些劳什子。

Babel官网提供了每一种环境下的配置文档：https://babeljs.io/docs/setup

![Using Babel](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbj5l51iuj211x0lbgmw.jpg)

> **注意**：上面的`In the browser`是将babel转换工作放在浏览器端执行，转换工作会花费大量的时间，所以这种方式只适合开发环境(实际上开发环境我们基本也不用它)，并不适用于生产环境。

我们实际开发大多使用构建系统帮我们做一些打包压缩的工作。

这里我以webpack为例介绍一下“**如何在webpack中使用babel**”

> 实际上都是傻瓜教程，相当简单。

## 1. 安装babel

```shell
npm install --save-dev babel-loader babel-core
```

## 2. 在webpack中配置babel-loader

```javascript
module: {
  rules: [
    { test: /\.js$/, exclude: /node_modules/, loader: "babel-loader" }
  ]
}
```

## 3. 创建.babelrc配置文件

根据需要安装预设，比如这里安装`presets-env`预设

```shell
npm install babel-preset-env --save-dev
```

在`.bashrc`中配置这些预设：

```json
{
  "presets": ["env"]
}
```

