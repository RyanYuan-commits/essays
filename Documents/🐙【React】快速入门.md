## 创建和嵌套组件
==组件==：返回标签的 JavaScript 函数。
```js
function MyButton() {  
	return (  
		<button>我是一个按钮</button>  
	);  
}
```

组件的嵌套
```js
export default function MyApp() {  
	return (  
		<div>  
			<h1>欢迎来到我的应用</h1>  
			<MyButton />  
		</div>  
	);  
}
```

## 使用 JSX 编写
JSX（JavaScript XML）是一种 JavaScript 的 **语法扩展**，它允许你在 JavaScript 代码中编写类似 XML 的结构。JSX 主要用于 React 库中，它并不是一种新的语言，而是一种语法糖，最终会被 Babel 等工具编译成纯 JavaScript 代码。

JSX 比 HTML 更加严格。你必须闭合标签，如 `<br />`。你的组件也不能返回多个 JSX 标签。你必须将它们包裹到一个共享的父级中，比如 `<div>...</div>` 或使用空的 `<>...</>` 包裹：
```js
function App() {
  return (
    <>
      <div> {user.name} </div>
    </>
  )
}
```

## 添加样式
在 React 中，你可以使用 `className` 来指定一个 CSS 的 class。它与 HTML 的 [`class`](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Global_attributes/class) 属性的工作方式相同：
```js
<img className="avatar" />
```

然后，你可以在一个单独的 CSS 文件中为它编写 CSS 规则：
```css
.avatar {
	border-radius: 50%;
}
```

React 并没有规定你如何添加 CSS 文件。最简单的方式是使用 HTML 的 [`<link>`](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/link) 标签。如果你使用了构建工具或框架，请阅读其文档来了解如何将 CSS 文件添加到你的项目中。

## 显示数据
JSX 会让你把标签放到 JavaScript 中。而大括号会让你 “回到” JavaScript 中，这样你就可以从你的代码中嵌入一些变量并展示给用户。例如，这将显示 `user.name`：
```js
return (
  <h1>
    {user.name}
  </h1>
);
```
你还可以将 JSX 属性 “转义到 JavaScript”，同样也是使用大括号。
例如，`className="avatar"` 是将 `"avatar"` 字符串传递给 `className`，作为 CSS 的 class。但 `src={user.imageUrl}` 会读取 JavaScript 的 `user.imageUrl` 变量，然后将该值作为 `src` 属性传递：
```js
return (
  <img
    className="avatar"
    src={user.imageUrl}
  />
);
```

## 条件渲染
React 没有特殊的语法来编写条件语句，因此你使用的就是普通的 JavaScript 代码。例如使用 [`if`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/if...else) 语句根据条件引入 JSX：
```js
let content;
if (isLoggedIn) {
  content = <AdminPanel />;
} else {
  content = <LoginForm />;
}
return (
  <div>
    {content}
  </div>
);
```
还可以用 ? 条件运算符来处理，当不需要 else 分支的时候，还有更简单的写法：
```java
<div>
  {isLoggedIn ? (
    <AdminPanel />
  ) : (
    <LoginForm />
  )}
</div>

<div>  
	{isLoggedIn && <AdminPanel />}  
</div>
```

## 渲染列表
你将依赖 JavaScript 的特性，例如 [`for` 循环](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/for) 和 [array 的 `map()` 函数](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/map) 来渲染组件列表。
假设你有一个产品数组：
```js
const products = [  
	{ title: 'Cabbage', id: 1 },  
	{ title: 'Garlic', id: 2 }, 
	{ title: 'Apple', id: 3 },
];
```
在你的组件中，使用 `map()` 函数将这个数组转换为 `<li>` 标签构成的列表:
```js
const listItems = products.map(product => 
	<li key={product.id}>    
		{product.title}  
	</li>
);
return (<ul> {listItems} </ul>);
```
注意， `<li>` 有一个 `key` 属性。对于列表中的每一个元素，你都应该传递一个字符串或者数字给 `key`，用于在其兄弟节点中唯一标识该元素。通常 key 来自你的数据，比如数据库中的 ID。
如果你在后续插入、删除或重新排序这些项目，React 将依靠你提供的 key 来思考发生了什么。

## 响应事件
你可以通过在组件中声明 **事件处理** 函数来响应事件：
```js
function MyButton() {
  function handleClick() {
    alert('You clicked me!');
  }

  return (
    <button onClick={handleClick}>
      点我
    </button>
  );
}
```
注意，`onClick={handleClick}` 的结尾没有小括号！不要 **调用** 事件处理函数：你只需 **把函数传递给事件** 即可。当用户点击按钮时 React 会调用你传递的事件处理函数。
## 更新界面
通常你会希望你的组件 “记住” 一些信息并展示出来，比如一个按钮被点击的次数。要做到这一点，你需要在你的组件中添加 **state**。

首先，从 React 引入 [`useState`](https://zh-hans.react.dev/reference/react/useState)：
```js
import { useState } from 'react';
```

现在你可以在你的组件中声明一个 **state 变量**：
```js
function MyButton() {
	const [count, setCount] = useState(0);
// ...
```
你将从 `useState` 中获得两样东西：
1. 当前的 state（`count`）
2. 用于更新它的函数（`setCount`）。
你可以给它们起任何名字，但按照惯例会像 `[something, setSomething]` 这样为它们命名。
## 使用 HOOK
以 use 开头的函数成为 HOOK，`useState` 是 React 提供的一个内置 Hook。你可以在 [React API 参考](https://zh-hans.react.dev/reference/react) 中找到其他内置的 Hook。你也可以通过组合现有的 Hook 来编写属于你自己的 Hook。
Hook 比普通函数更为严格。你只能在你的组件（或其他 Hook）的 **顶层** 调用 Hook。如果你想在一个条件或循环中使用 `useState`，请提取一个新的组件并在组件内部使用它。
## 组件之间共享数据
将 `MyButton` 的 **state 上移到** `MyApp` 中：
```js
import { useState } from 'react';
import './App.css'

export default function App() {
  const [num, setNum] = useState(0)

  function handleClick() {
    setNum(num + 1)
  }
  

  return (
    <div>
      <h1>共同更新的计数器</h1>
      <MyButton count={count} onClick={handleClick} />
      <MyButton count={count} onClick={handleClick} />
    </div>
  )
}

function MyButton({ count, onClick }) {
  return (
    <button onClick={onClick}>我是一个按钮, num = {count}</button>
  );
}
```