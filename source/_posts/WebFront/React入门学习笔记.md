---
title: React入门学习笔记
date: 2018-03-15
categories: 前端
---

## React介绍

谷歌大法，一搜一大把

## React环境安装

安装`react`，`react-dom`模块：

```shell
cnpm install react react-dom --save
```

因为react中使用了JSX语法，所以需要`babel`进行转换：

```shell
cnpm install babel-preset-react babel-core --save-dev
```

如果项目使用ES6语法，还需要一个ES6转ES5的`preset`：

> [react官方](https://reactjs.org/)已极力推荐使用ES6语法编写应用，官方的demo都是基于ES6的。

```shell
cnpm install babel-preset-es2015 --save-dev
```

> React团队提供了一个[`create-react-app`](https://github.com/facebook/create-react-app/blob/master/README.md#getting-started)的工具，只需在命令行输入`create-react-app <app-name>`，工具会帮你安装`react`，`react-dom`，`react-script`并生成项目的基本框架。

### webpack配置

在安装完成后我们首先编写一段react语法的代码

main.js:

```javascript
import React from 'react'
import ReactDOM from 'react-dom'

ReactDOM.render(
	<h1>Hello react!</h1>,
	document.getElementById("app")
)
```

index.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<title>Document</title>
</head>
<body>
	<div id="app">hello vue!</div>
</body>
</html>
```

全局安装`webapck`:

`cnpm install webpack -g`

在项目根目录安装`webpack`:

`cnpm install webpack --save-dev`

然后在配置`webpack.config.js`:

```javascript
var webpack = require('webpack');

module.exports = {
    entry: './index.jsx',
    output: {
        filename: 'bundle.js'
    },
    module: {
        rules: [{
            test: /\.js[x]?$/,
            exclude: /node_modules/,
            loader: 'babel-loader'
        }]
    }
};
```

然后执行`webpack`打包就可以了

如果要热更新的话可以再安装:

`cnpm install webpack-dev-server --save`

然后执行：`webpack-dev-server --progress --inline`来启动热更新的服务器

当然也可以将这个命令写到`package.json`的`scripts`里,然后通过`npm run dev`来启动

```json
"scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "dev": "webpack-dev-server --progress --inline",
    "build": "webpack --env production"
 }
```

或者一次性安装所有的：

`cnpm install babel-core babel-loader babel-preset-es2015 babel-preset-react react react-dom webpack webpack-dev-server --save`

## React组件

组件化开发是react开发中最重要的功能，可以简单的认为react就是开发组件的.

组件系统react的重要概念，因为它是一种抽象，允许我们使用小型、自包含和通常可复用的组件构建大型应用。仔细想想，几乎任意类型的应用界面都可以抽象为一个组件树：

![组件树](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbj6arz3vj21320f4t8v.jpg)

### 定义react组件

编写一个`Header`组件：

```jsx
import React from 'react'

// 编写一个Header类继承自React.Component类
// 类名首字母要大写：JSX解析时会把小写开头的当做HTML标签解析
export default class Header extends React.Component{
	render() {
        // return后面的dom定义不能换行
		return <header>
				<h1>我是头部</h1>
			   </header>
	}
}
```

**render方法是React组件必要的方法**。render方法中使用JSX语法定义DOM结构。

> 如果组件比较简单，还可以使用函数式声明的组件：
>
> ```jsx
> function Header() {
>   return <header>
>     <h1>我是头部</h1>
>   </header>
> }
> ```
>
> 具体使用详见官方文档：https://reactjs.org/docs/components-and-props.html#functional-and-class-components

在`main.js`中引用上面定义的组件：

```jsx
import React from 'react'
import ReactDOM from 'react-dom'
// 引入组件
import Header from './components/header'

class App extends React.Component{
	render() {
        // 要换行得加括号
		return (
            // 只允许有一个顶层元素
            // 顶层元素内部可以嵌套多个组件
            <div>
				<Header/>
				<main>页面的主体内容</main>
            </div>
			)
	}
}

// 定义app组件渲染的位置，相当于入口
ReactDOM.render(<App/>, document.getElementById("app"))
```

> render方法中只允许有一个顶层元素返回，不过React提供了一个[`React.Fragment`](https://reactjs.org/docs/react-api.html#reactfragment)可以让你嵌套多个子元素
>
> ```jsx
> render() {
>   return (
>     <React.Fragment>
>       Some text.
>       <h2>A heading</h2>
>     </React.Fragment>
>   );
> }
> ```
>

## JSX

**JSX=JS+XHTML**，其中XHTML要求HTML格式符合XML标准，不能像写HTML一样随意：标签闭合，空标签`<tag/>`。

**JSX基本的解析规则**：遇到 HTML 标签（以 `<` 开头），就用 HTML 规则解析；遇到代码块（以 `{` 开头），就用 JavaScript 规则解析。上面代码的运行结果如下。

其实JSX本质上会创建[虚拟DOM(virtual-dom)](https://github.com/Matt-Esch/virtual-dom)对象。

比如上面的Header定义会被转换成：

```jsx
export default class Header extends React.Component{
	render() {
        // return后面的dom定义不能换行
		return <header className='header'>
				<h1>我是头部</h1>
			   </header>
	}
}
==>
var Header = React.createClass({
  render: function() {
    return React.createElement(
                 'header',
                 // js中通过className获取元素的class属性
                 { className: 'header'},
                 React.createElement('h1',null,'我是头部')
                              );
  }
});
==>
<header class='header'>
  <h1>我是头部</h1>
</header>
```

React.createClass就是用来创建React组件的。

> React.createClass是ES5创建组件的方式，新版本的React推荐使用继承React.Component的方式创建组件。如果不想使用ES6，需要添加`create-react-class`模块依赖，具体可以参考官方文档：[React Without ES6](https://reactjs.org/docs/react-without-es6.html)

[React.createElement](https://reactjs.org/docs/react-api.html#createelement)就是用来创建**虚拟DOM对象**的。

> **注意**：上面的header元素不通过class而使用className来引用CSS类，不仅仅因为class与ES6的关键字冲突；最主要的原因React将标签中的所有属性最终会作为一个对象传给`React.createElement`方法，JS中通过`className`(字符形式)或`classList`(数组形式)属性来操作class，显然用className更合适。

## 虚拟DOM

> 参考React官网的文档：https://reactjs.org/docs/faq-internals.html#what-is-the-virtual-dom

Web App性能问题主要出在 DOM 对象的操作上，比如读写，创建，插入等等。之所以原生 DOM 性能低，是因为 DOM 的规范迫使浏览器在实现的时候为每一个 DOM 元素添加了非常多的属性，然而这其中很多我们都用不到。

具体到 [React 主要是做了两点](https://reactjs.org/docs/optimizing-performance.html)：

其一是 VirtualDOM，一个很简化的虚拟文档对象模型系统，你可以操作类似 DOM 的对象，但是非常轻量化（这就是为啥 JSX 看起来是在 JS 代码里写 XML 的缘故，这是一层语法糖，方便开发者编写模板，但实际上还是 JS 对象，而不是真实的 DOM 对象）；

其二则是当数据变化的时候，不直接去修改 DOM（因为变化的只是个别属性，但是修改 DOM 往往却要替换一整个 DOM 对象），而是先用[dom diff算法](https://segmentfault.com/a/1190000000606216)(简单的理解成`git diff`这样的命令)比较前后的差异，最后只把变化的部分一次性应用到真实的 DOM 树上去。

> 可以联想到游戏开发中经常提到的“双缓冲”机制，两者有异曲同工之妙

## 使用组件对象的`props`访问标签属性

```jsx
class Header extends React.Component{
	render() {
		return <header className='header'>
				<h1>{this.props.text}</h1>
			   </header>
	}
}

ReactDOM.render(<Header text='我是头部'/>, document.getElementById("app"));
```

## `this.props.children`访问标签子元素

props中还有一个特殊的属性`children`用于访问标签中的子元素。

```jsx
class List extends React.Component{
	render() {
		return <ol>{
          React.Children.map(this.props.children, function(child){
            return <li>{child}</li>;
          })
        }
        </ol>
	}
}

ReactDOM.render(<List>
    <span>item1</span>
    <span>item2</span>
  </List>,
  document.getElementById("app"));
```

最终会被渲染成：

```html
<ol>
  <li><span>item1</span></li>
  <li><span>item2</span></li>
</ol>
```

> 因为标签子元素个数不确定：没有子元素，`children=undefined`；一个子元素，值为object类型；多个子元素，就是array类型。
>
> 为了方便操作React在[`React.Children`](https://reactjs.org/docs/react-api.html#reactchildren)中提供了一套工具方法，方便操作子元素。

## 使用`defaultProps`定义默认的props

如果标签中没有指定属性值，`this.props`会取出`undefined`值，可以在组件的[`defaultProps`](https://reactjs.org/docs/react-component.html#defaultprops)中定义默认的属性值。

```jsx
class Header extends React.Component{
	render() {
		return <header className='header'>
				<h1>{this.props.text}</h1>
			   </header>
	}
}
Header.defaultProps = {
  text: '默认标题文本'
}

ReactDOM.render(<Header/>, document.getElementById("app"));
```

> 如果不使用ES6，在调用createReactClass时，需要在对象中定义一个`getDefaultProps`方法。详见：https://reactjs.org/docs/react-without-es6.html#declaring-default-props

## 对props中的属性进行类型检查

> 参考：https://reactjs.org/docs/typechecking-with-proptypes.html

为了限制组件中属性的类型，需要`React.PropTypes`进行类型检查。

> 从React v15.5开始，`React.PropTypes`被移进了[`prop-types`](https://www.npmjs.com/package/prop-types)库。

```jsx
import PropTypes from 'prop-types';

class Header extends React.Component{
	render() {
		return <header className='header'>
				<h1>{this.props.text}</h1>
			   </header>
	}
}
Header.propTypes = {
  // 告诉React：title必须是string类型的，而且是必填属性
  text: PropTypes.string.isRequired
}

ReactDOM.render(<Header text='文本'/>, document.getElementById("app"));
```

## 获取真实的DOM节点

> 参考：https://reactjs.org/docs/refs-and-the-dom.html

前面说到React的render方法创建的是虚拟DOM对象，它并不是真实的 DOM 节点，而是存在于内存之中的一种数据结构，当虚拟DOM更新到浏览器的DOM后，我们要怎么操作真实的DOM节点呢？这时就要用到 `ref` 属性。

### String refs

先看老版本API中的`ref`怎么使用：

```jsx
class MyComponent extends React.Component{
  handleClick() {
    // 使用refs属性访问元素
    this.refs.textInput.focus();
  },
  render() {
    return (
      <div>
        <input type="text" ref="textInput" />
        <input type="button" value="点击按钮,让文本框获取焦点" onClick={this.handleClick} />
      </div>
    );
  }
});

ReactDOM.render(
  <MyComponent />,
  document.getElementById('example')
);
```

传统的方式要将在元素中定义`ref`属性，相当于给元素定义了一个id，然后组件中可以在`this.refs`属性根据`ref`指定的id获取标签元素。

### ref callback

新版本的不再使用字符引用的方式，[这里](https://github.com/facebook/react/pull/8333#issuecomment-271648615)有关于两者的讨论。

```jsx
class MyComponent extends React.Component{
  handleClick() {
    // 下面的ref回调方法中将this.textInput指向了input元素
    this.textInput.focus();
  },
  render() {
    return (
      <div>
        <input type="text" ref={ (input)=>{this.textInput=input} } />
        <input type="button" value="点击按钮,让文本框获取焦点" onClick={this.handleClick} />
      </div>
    );
  }
});

ReactDOM.render(
  <MyComponent />,
  document.getElementById('example')
);
```

## setState方法改变组件状态`this.state`

UI组件避免不了与用户的交互，有交互就避免不了对组件的修改。React将组件看成一个状态机，**一开始有一个初始状态**，一旦有状态的改变，就会触发重新渲染。

```jsx
class LikeButton extends React.Component {
  constructor(props) {
    // 调用父类的构造方法,父类可能也有自己的组件状态
    super(props);
    // 初始化组件状态
    this.state = {liked: false};
  }
  handleClick(event) {
    // 点击时间触发时，修改控件状态，控件重新渲染
    this.setState({liked: !this.state.liked});
  },
  render() {
    var text = this.state.liked ? 'like' : 'haven\'t liked';
    return (
      <p onClick={this.handleClick}>
        You {text} this. Click to toggle.
      </p>
    );
  }
}
```

> **注意**：
>
> * 不要通过`this.state.comment = 'Hello';`这种方式直接修改控件状态，因为这样不会触发控件重新渲染。而应该使用`setState`修改状态。
> * `this.state = XXX`这种赋值操作只能在构造函数中初始化时使用。

> 在ES5中使用React时，需要在createReactClass方法中的提供**`getInitialState`**方法返回初始状态。
>
> 详见官方文档：https://reactjs.org/docs/react-without-es6.html#setting-the-initial-state

### 根据前一个状态计算后续状态

因为setState是异步更新状态，我们不应该直接依赖当前的state值计算下一个状态：

```javascript
// Wrong
this.setState({
  counter: this.state.counter + this.props.increment,
});

// Correct
this.setState((prevState, props) => ({
  counter: prevState.counter + props.increment
}));
```

## 组件生命周期

> 参考：https://reactjs.org/docs/state-and-lifecycle.html
>
> https://reactjs.org/docs/react-component.html#the-component-lifecycle

每个组件都有几个**生命周期方法**，可以重写这几个方法，React在特定的时候会回调这些方法。

这些生命周期方法有些特征：

**前缀为*will*的方法表示某些事件即将发生；前缀为*did*的方法表示某些事件已经发生**

![组件生命周期](https://sfault-image.b0.upaiyun.com/104/523/1045236351-57ce24c128a2a_articlex)

当组件实例被创建，再到组件被插入到DOM树中，这个过程会触发组件的以下方法：

- [`constructor()`](https://reactjs.org/docs/react-component.html#constructor)
- [`componentWillMount()`](https://reactjs.org/docs/react-component.html#componentwillmount)
- [`render()`](https://reactjs.org/docs/react-component.html#render)
- [`componentDidMount()`](https://reactjs.org/docs/react-component.html#componentdidmount)

当`props`或`state`被修改导致组件更新，组件渲染的时候会回调下面这些方法：

- [`componentWillReceiveProps()`](https://reactjs.org/docs/react-component.html#componentwillreceiveprops)
- [`shouldComponentUpdate()`](https://reactjs.org/docs/react-component.html#shouldcomponentupdate)
- [`componentWillUpdate()`](https://reactjs.org/docs/react-component.html#componentwillupdate)
- [`render()`](https://reactjs.org/docs/react-component.html#render)
- [`componentDidUpdate()`](https://reactjs.org/docs/react-component.html#componentdidupdate)

当组件从DOM树中移除的时候，会触发componentWillUnmount方法：

- [`componentWillUnmount()`](https://reactjs.org/docs/react-component.html#componentwillunmount)

在组件渲染、生命周期回调或者构造子组件的过程中出现异常，会触发组件的componentDidCatch方法：

- [`componentDidCatch()`](https://reactjs.org/docs/react-component.html#componentdidcatch)

> 方法的具体介绍可以点击相应链接看官方文档的介绍

## 一个简单的时钟组件例子

*component/TimerClock.jsx*

```jsx
import React from 'react'

export default class TimerClock extends React.Component{
	constructor(props) {
    	super(props);
 		this.state = {
 			date: new Date(),
 			running: true,
 			title: '点击暂停'
 		};
 	}
 	// 在组件挂载到DOM树时,开启定时器
	componentDidMount() {
		this.start();
	}
	// 在组件从DOM树卸载时,关闭定时器
	componentWillUnmount() {
		this.stop();
	}
	toggle() {
		if(this.state.running) {
			this.stop();
		} else {
			this.start();
		}
	}
	start() {
		this.timerID = setInterval(() => this.tick(), 1000);
		this.setState({
			running: true,
			title: '点击暂停'
		});
	}
	stop() {
    	clearInterval(this.timerID);
    	this.setState({
			running: false,
			title: '点击继续'
		});
	}
	// 每秒重新设置组件状态
	tick() {
		this.setState({
			date: new Date()
		});
	}
	render() {
		return <div onClick={() => this.toggle()} title={this.state.title}>
			<span>当前时间为:</span>
			<span>{this.state.date.toLocaleTimeString()}</span>
		</div>
	}
}
```

*index.jsx*

```jsx
import React from 'react'
import ReactDOM from 'react-dom'
import TimerClock from './component/TimerClock.jsx'

ReactDOM.render(<TimerClock />, document.getElementById('react-container'));
```

*index.html*

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<title>时钟组件</title>
</head>
<body>
	<div id="react-container"></div>
	<script type="text/javascript" src="bundle.js"></script>
</body>
</html>
```

*webpack.config.js*

```javascript
var webpack = require('webpack');

module.exports = {
    entry: './index.jsx',
    output: {
        filename: 'bundle.js'
    },
    module: {
        rules: [{
            test: /\.js[x]?$/,
            exclude: /node_modules/,
            loader: 'babel-loader'
        }]
    }
};
```

*package.json*

```json
{
  "name": "my-component",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": ""
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "react": "^16.2.0",
    "react-dom": "^16.2.0"
  },
  "devDependencies": {
    "babel-core": "^6.26.0",
    "babel-loader": "^7.1.4",
    "babel-preset-env": "^1.6.1",
    "babel-preset-react": "^6.24.1",
    "webpack": "^4.1.1"
  }
}
```



参考：

https://reactjs.org/docs/react-api.html

http://www.ruanyifeng.com/blog/2015/03/react.html