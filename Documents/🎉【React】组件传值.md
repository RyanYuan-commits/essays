React 组件使用 _props_ 来互相通信。每个父组件都可以提供 props 给它的子组件，从而将一些信息传递给它。Props 可能会让你想起 HTML 属性，但你可以通过它们传递任何 JavaScript 值，包括对象、数组和函数。

Props 是你传递给 JSX 标签的信息，比如 className、src、alt、width 和 height 都是一些可以传递给 img 标签的 props
```js
function Avatar() {
  return (
    <img
      className="avatar"
      src="https://i.imgur.com/1bX5QH6.jpg"
      alt="Lin Lanying"
      width={100}
      height={100}
    />
  );
}
```

## 向组件传递 Props
首先，将一些 props 传递给 `Avatar`。例如，让我们传递两个 props：`person`（一个对象）和 `size`（一个数字）：
```js
export default function Profile() {
  return (
    <Avatar
      person={{ name: 'Lin Lanying', imageId: '1bX5QH6' }}
      size={100}
    />
  );
}
```

在子组件中读取 Props
```js
function Avatar({ person, size }) {
  // 在这里 person 和 size 是可访问的
}
```

在声明 props 时， **不要忘记 `(` 和 `)` 之间的一对花括号 `{` 和 `}`** ：
```js
function Avatar({ person, size }) {
  // ...
}
```
这种语法被称为 [“解构”](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment#Unpacking_fields_from_objects_passed_as_a_function_parameter)，等价于于从函数参数中读取属性：

```js
function Avatar(props) {
  let person = props.person;
  let size = props.size;
  // ...
}
```

## 给 Props 指定默认值
如果你想在没有指定值的情况下给 prop 一个默认值，你可以通过在参数后面写 `=` 和默认值来进行解构：
```js
function Avatar({ person, size = 100 }) {
  // ...
}
```

## 使用 JSX 展开语法传递 Props
有时候，传递 props 会变得非常重复：
```js
function Profile({ person, size, isSepia, thickBorder }) {
  return (
    <div className="card">
      <Avatar
        person={person}
        size={size}
        isSepia={isSepia}
        thickBorder={thickBorder}
      />
    </div>
  );
}
```

重复代码没有错（它可以更清晰）。但有时你可能会重视简洁。一些组件将它们所有的 props 转发给子组件，正如 `Profile` 转给 `Avatar` 那样。因为这些组件不直接使用他们本身的任何 props，所以使用更简洁的“展开”语法是有意义的：
```js
function Profile(props) {
  return (
    <div className="card">
      <Avatar {...props} />
    </div>
  );
}
```
这会将 `Profile` 的所有 props 转发到 `Avatar`，而不列出每个名字。
**请克制地使用展开语法。** 如果你在所有其他组件中都使用它，那就有问题了。 通常，它表示你应该拆分组件，并将子组件作为 JSX 传递。 接下来会详细介绍！

## 传递 JSX
当你将内容嵌套在 JSX 标签中时，父组件将在名为 `children` 的 prop 中接收到该内容。
例如，下面的 `Card` 组件将接收一个被设为 `<Avatar />` 的 `children` prop 并将其包裹在 div 中渲染：
```jsx
import Avatar from './Avatar.js';

function Card({ children }) {
  return (
    <div className="card">
      {children}
    </div>
  );
}

export default function Profile() {
  return (
    <Card>
      <Avatar
        size={100}
        person={{ 
          name: 'Katsuko Saruhashi',
          imageId: 'YfeOqp2'
        }}
      />
    </Card>
  );
}
```