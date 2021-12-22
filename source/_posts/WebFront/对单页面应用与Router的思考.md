---
title: 对单页面应用与Router的一些思考
date: 2018-04-26
categories: 前端
---

昨晚鑫哥到我宿舍聊天，聊了很多，从各自公司用的技术到杭州和深圳的房价，从后端技术到前端技术，一直聊到凌晨一点多，直到鑫哥被室友“驱逐”出去。中间有一段聊到公司用到的React，提到了单页面应用和Router，觉得思路很好有必要写个笔记记下来。

## 最开始的网页<i id='_top_'></i>

早期的网页都是一个个独立的html页面，通过`a`标签从这个页面跳转到另外一个页面。但是同一个网站中的两个页面很多内容都是相同的，比如页头、页脚、导航栏和主菜单等。

两次网络请求得到的数据中很大一部分都是重复的，不管这两个页面是完全静态的（不好维护），还是由后台动态语言程序生成的，都会浪费不小的带宽。

## frame布局

为了便于维护，同时为了点击左边的导航栏只有右边的内容部分更新，所以有一部分网站用frameset布局。把公共的部分抽成单独的html，在frameset中引入各个html。这种布局方式在很多老的管理系统中还可以见到。

> 不过[frameset在HTML5标准中被弃用了](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/frameset)，只剩下一个[iframe还可以用](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/iframe)。

![frameset布局](http://tva1.sinaimg.cn/large/bda5cd74gy1fqq374epnej211x0hv0xf.jpg)

## ajax带来的web2.0时代

IE5 通过 ActiveX 来实现了Ajax后， Mozilia，Safari，Opera 相继以 XMLHttpRequest 来实现 Ajax。有了Ajax，用户交互再也不用刷新整个页面了，单页面应用(SPA:single page web application)也初具雏形。

![ajax](http://tva1.sinaimg.cn/large/bda5cd74gy1fqq3m7qa9hg20mn09wade.gif)

有了ajax技术，但是关于“怎么用好ajax”这个问题，很长一段时间都是在探索中。

### 请求的数据如何渲染

#### 1. 最原始的手动拼接字符串，然后append到dom中。

这种方式可读性差，不方便维护。

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <script src="https://cdn.bootcss.com/jquery/1.12.4/jquery.min.js"></script>
    <script src="https://cdn.bootcss.com/lodash.js/4.17.5/lodash.core.min.js"></script>
</head>
<body>
    <ul id='container'></ul>
    <script>
    // $.get('https://api.github.com/search/repositories?q=javascript&sort=stars&order=desc', function(data) {
    //     for (var item of data.items) {
    //         $('#container')
    //             .append('<li>' +
    //                 '<h4><a href=' + item.html_url + '>' + item.name + '</a></h4>' +
    //                 '<small>' + item.description + '</small>' +
    //                 '</li>');
    //     }
    // });

    // 就像游戏开发中要求双缓冲一样，前端也要避免频繁的操作DOM，而应该一次性地将内容append到DOM中。
    $.get('https://api.github.com/search/repositories?q=javascript&sort=stars&order=desc', function(data) {
        var innerHTML = '';
        for (var item of data.items) {
            innerHTML = innerHTML + '<li>' +
                '<h4><a href=' + item.html_url + '>' + item.name + '</a></h4>' +
                '<small>' + _.escape(item.description) + '</small>' +
                '</li>';
        }
        $('#container').append(innerHTML);
    });
    </script>
</body>
</html>
```

> Vue.js和React.js都使用Virtual DOM的思想尽量将DOM操作的次数减少到最小。代码中也不再直接操作DOM了，jQuery这种直接操作DOM的时代已经成了过去式。

#### 2. 前端模板引擎

为了让HTML模板可读性更强，实现**数据与视图分离**。前端衍生出了N多种模板引擎，从[jQuery.tmpl](https://github.com/BorisMoore/jquery-tmpl)小插件，到独立的库[JsRender](https://github.com/borismoore/jsrender)、[JsViews](http://www.jsviews.com/)，到和后端JSP语法贼像的[EJS](https://github.com/mde/ejs)，再到大名鼎鼎的[mustache.js](https://github.com/janl/mustache.js)和[handlebars](https://github.com/wycats/handlebars.js)...，还有国内的[baiduTemplate](https://github.com/BaiduFE/BaiduTemplate)，[art-template](https://github.com/aui/art-template)，[template.js](https://github.com/yanhaijing/template.js)，[Juicer](https://github.com/PaulGuo/Juicer/)...，模板引擎百花齐放。

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <script src="https://cdn.bootcss.com/mustache.js/2.3.0/mustache.js"></script>
    <script src="https://cdn.bootcss.com/axios/0.18.0/axios.min.js"></script>
</head>
<body>
    <div id='container'></div>
    <script id="items" type="x-tmpl-mustache">
        <ul>
            {{#items}}
            <li>
                <h4>
                	<a href='{{html_url}}'>{{name}}</a>
                	<sub>star:{{stargazers_count}},fork:{{forks_count}}</sub>
      			</h4>
                <small>{{description}}</small>
            </li>
            {{/items}}
        </ul>
    </script>
    <script>
    var template = document.getElementById('items').innerHTML;
    axios.get('https://api.github.com/search/repositories?q=javascript&sort=stars&order=desc').then(function(response) {
        var rendered = Mustache.render(template, response.data);
        document.getElementById('container').innerHTML = rendered;
    });
    </script>
</body>
</html>
```

#### 3. 模块化组件

当应用变得越来越大的时候，页面中就会出现一大堆模板片段以及ajax的代码，很明显不利于维护，所以很有必要将页面中的元素抽成模块化的组件，组件内部可以嵌套组件。然后就形成了一棵组件树：

![组件树](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbj6arz3vj21320f4t8v.jpg)

```JSX
import React from "react";
import _ from "lodash";
import axios from "axios";

export class List extends React.Component {

    constructor(props) {
        super(props);
        this.state = {};
    }

    componentDidMount() {
        axios.get('https://api.github.com/search/repositories?q=javascript&sort=stars&order=desc')
            .then(response => this.setState({data: response.data}))
            .catch(err => console.log(err));
    }

    render() {
        let {data} = this.state;
        return data ? (
            <ul>
                { _.map(_.get(data, "items"), (item) => <ListItem item={item}/>)}
            </ul>
        ) : (
            <p>正在加载中...</p>
        );
    }
}

const ListItem = ({item}) => {
    return (
        <li>
            <h4>
                <a href="{item.html_url}">{item.name}</a>
                <sub>star:{item.stargazers_count}, fork:{item.forks_count}</sub>
            </h4>
            <small>{item.description}</small>
        </li>
    );
}
```

> 上面是用React写的例子，在我的github上可以找到完整的代码：https://github.com/holmofy/react-demos/blob/master/my-http-request-component/component/List.jsx
>
> 我还写了一个Vue版本的例子：https://github.com/holmofy/Vue-Learn/tree/master/my-http-request-component
>
> 其实不管是[Vue](https://vuejs.org/)、[Angular](https://angularjs.org/)亦或者[Ember.js](https://www.emberjs.com/)等，他们都是以组件的形式开发。

> 前端的[HTML](https://en.wikipedia.org/wiki/Web_Components)([Web Component](https://www.w3.org/wiki/WebComponents/))、CSS([CSS Module](https://github.com/css-modules/css-modules))、JS([CommonJS](https://en.wikipedia.org/wiki/CommonJS)，[AMD](https://en.wikipedia.org/wiki/Asynchronous_module_definition))都已经往模块化的方向发展

###  页面路由

由于页面中大部分元素是ajax请求后渲染出来的，但**由于ajax请求不会改变地址栏，无法保持页面的状态**。用户不能把页面的某个状态以url的方式分享给其他人，比如jQuery-EasyUI官方文档就是这样的例子，我想把droppable的文档发给了同学，他打开页面后还得在插件列表里再去找droppable。

> jQuery-EasyUI会以tab的方式展示ajax请求HTML片段。EasyUI体积也比较大(加上所有控件)，比较适合做公司内部的一些管理系统。拿来做后台系统，像上面说的分享url的情况应该会很少。

![jQuery-EasyUI](http://tva1.sinaimg.cn/large/bda5cd74gy1fqqbdky9yjg20zy0j8te6.gif)


简单的说，**路由就是把页面的状态保存在url中**。

#### 1. hash路由

我们知道`window.location`可以操作地址栏修改url，但是修改了[Location](https://developer.mozilla.org/en-US/docs/Web/API/Location)就会向服务器发送请求并发生页面跳转。好在[Location里面有个hash属性](https://developer.mozilla.org/en-US/docs/Web/API/HTMLHyperlinkElementUtils/hash)修改了并不会发生页面跳转。这个hash就是我们常说的**锚点**。通常我们用锚点来实现页面内的跳转(比如[回到顶部](#_top_))：`<a href='#top'>回到顶部</a>`。现在我们用它表示页面当前的状态：`https://example.com#/entry/path`。比如阿里云就是用hash路由。

![hash路由](http://tva1.sinaimg.cn/large/bda5cd74gy1fqqd5oeltng21020j610d.gif)

#### 2. history.push带来的新路由

HTML5标准新增的[History API](https://developer.mozilla.org/en-US/docs/Web/API/History_API)，为[History](https://developer.mozilla.org/en-US/docs/Web/API/History)(`window.history`)添加了两个新方法`pushState`和`replaceState`，让我们可以操作浏览器历史。

pushState方法可以让url改变，但是并不会导致浏览器跳转。

从某种意义上讲，调用`pushState()`与`window.location = "#foo"`类似，因为两者都会**改变地址栏而不发送网络请求**并且创建一条新的浏览历史记录。

腾讯云用的就是push路由。

![push路由](http://tva1.sinaimg.cn/large/bda5cd74gy1fqqdw8tm5hg21020j7qd2.gif)

> jQuery有一个插件[jQuery-pjax](https://github.com/defunkt/jquery-pjax)（pjax=pushState+ajax）就通过ajax请求html片段，并使用pushState修改地址栏。

**Vue有相应的[vue-router](https://router.vuejs.org)模块，React有[react-router](https://reacttraining.com/react-router/)、Angular有[angular-route](https://angular.io/guide/router)。路由也是单页面应用的一个重要组成部分**。

除了框架自己的路由模块外，还有一些第三方路由，如[Page.js](https://github.com/visionmedia/page.js) 和[Director](https://github.com/flatiron/director) 

参考资料：

https://developer.mozilla.org/en-US/docs/Web/API/Location

https://developer.mozilla.org/en-US/docs/Web/API/History_API

