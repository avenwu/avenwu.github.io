---
layout: post
title: "Fullstack Rect 笔记-JSX"
description: "Fullstack React 笔记-JSX"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2018-08-10-01.png
keywords: "jsx"
tags: [react]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2018-08-10-01.png)

## JSX书写Tips

JSX =》`JavaScript Syntax Extension` =》JS语法糖扩展

使用JSX可以大幅提高代码书写的简洁性，特别是在多层级的组件嵌套书写是产生的层级问题。

* JSX中写注释

```jsx
{/*
  注释内容
*/}
```

* JSX Boolean属性

bool属性需要用`{}`包括，类似的，变量值也需要用`{}`包裹

```jsx
<input name='Name' disabled={true} />
```

* JSX 条件表达式

通过`&&`可选的输出内容，而不需要`？：`

```jsx
const renderAdminMenu = function() {
  return (<MenuLink to="/users">User accounts</MenuLink>)
}
const userLevel = this.props.userLevel; return (
  <ul>
    <li>Menu</li>
    {userLevel === 'admin' && renderAdminMenu()}
  </ul>
)
```

* JSX透传属性语法

JSX可以透传属性，已更简单书写来赋值属性`{...props}`

```jsx
{/* 常规写法，将msg和recipent赋值给Component组件的属性 */}
const props = {msg: "Hello", recipient: "World"}
<Component msg={"Hello"} recipient={"World"} />

<Component {...props} />
<!-- essentially the same as this: --> 
<Component msg={"Hello"} recipient={"World"} />
```

* class & className区分

JSX中使用className来指定class

```jsx
<!-- Same as <div class='box'></div> --> 
<div className='box'></div>

var cssNames = ['box', 'alert']
// and use the array of cssNames in JSX
(<div className={cssNames.join(' ')}></div>)
```

* classnames模块

支持class的动态控制，为true的class生效

```jsx
class App extends React.Component { 
  render() {
	const klass = classnames({
		box: true, // always apply the box class
		alert: this.props.isAlert, // if a prop is set 
         severity: this.state.onHighAlert, // with a state 
         timed: false // never apply this class
	});
	return (<div className={klass} />) 
  }
}
```

* 自定义属性

`data-anything`为html标准标签添加自定义的属性需要以`data-`开头，自定义组件则不受这个约束

```jsx
<div className='box' data-dismissible={true} />
<span data-highlight={true} />
```

```jsx
<Message dismissible={true} />
<Note highlight={true} />
```

## 高阶组件配置

* PropTypes属性类型

通过`prop-types`包可以对属性类型做限制，同时提供一个优良的说明

```jsx
class Map extends React.Component { 
  static propTypes = {
  	lat: PropTypes.number,
	lng: PropTypes.number,
  	zoom: PropTypes.number,
  	place: PropTypes.object,
  	markers: PropTypes.array,
};
const Map = React.createClass({ 
  propTypes: {
	lat: PropTypes.number, 
    lng: PropTypes.number // ...
  }, 
})
```

* props默认值

```jsx
class Counter extends React.Component { static defaultProps = {
    initialValue: 1
  };
// ...
};
<!-- 等价 -->
<Counter />
<Counter initialValue={1} />
```

* 全局上下文共享属性

通过全局的context可以实现属性数据共享，避免将数据从顶层组件一层层传递赋值，但是要`尽量避免滥用context`;

第一步：在父组件定义childContextTypes和getChildContext；

第二步：在子组件定义；

第三步：在子组件中通过this.context.xxx可以获取共享的数据；

```jsx
// 父组件
class Messages extends React.Component { // ...
	static childContextTypes = { 
      users: PropTypes.array, 
      userMap: PropTypes.object,
	};
	// ...
	getChildContext() { 
      return {
		users: this.getUsers(),
		userMap: this.getUserMap(), 
      };
	}
	// ...
}
// 子组件
class ThreadList extends React.Component { 
  	// ...
	static contextTypes = { 
      users: PropTypes.array,
	};
	// ...
}
class ChatWindow extends React.Component { // ...
	static contextTypes = { 
      userMap: PropTypes.object,
	};
	// ...
}

```

* 自定义嵌套组件

自定义的组件也可以是类似div这种支持内部嵌套组件；要实现这种效果，需要使用`this.props.chidren`

```jsx
const Newspaper = props => { 
	return (
    	<Container>
      		<Article headline="An interesting Article">
				Content Here
      		</Article>
    	</Container>
	); 
}

class Container extends React.Component { 
	//...
	render() {
		return (
			<div className='container'> 
				{this.props.children}
			</div>
		);
	}

}
```

* 子组件与map/forEach

将一个数组数据按照一定规则生成ReactElement经常用到map方法，类似的子组件也有map方法：`React.Children.map()`/`React.Children.forEach()`

```jsx
class Container extends React.Component { 
  static propTypes = {
    component: PropTypes.element.isRequired,
    children: PropTypes.element.isRequired
	//...
	renderChild = (childData, index) => { 
    	return React.createElement(
			this.props.component,
			{}, // <~ child props
			childData // <~ child's children
		) 
	}
	// ...
	render() { 
      return (
      <div className='container'>
        {React.Children.map(
			this.props.children,
			this.renderChild )
        }
	 </div> )
	} 
}
```


