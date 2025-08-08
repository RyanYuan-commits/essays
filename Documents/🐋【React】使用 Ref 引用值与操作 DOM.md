当你希望组件“记住”某些信息，但又不想让这些信息 触发新的渲染 时，你可以使用 `ref` 。
## 使用 ref 引用值
### 给你的组件添加 ref
```jsx
import { useRef } from 'react';
const ref = useRef(0);
```
`useRef` 返回的是这样的一个对象：
```jsx
{  
	current: 0 // 你向 useRef 传入的值  
}
```
你可以用 `ref.current` 属性访问该 ref 的当前值。这个值是有意被设置为可变的，意味着你既可以读取它也可以写入它。就像一个 React 追踪不到的、用来存储组件信息的秘密“口袋”。
### ref 和 react 的不同之处
| ref                                                  | state                                                                                         |
| ---------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| `useRef(initialValue)`返回 `{ current: initialValue }` | `useState(initialValue)` 返回 state 变量的当前值和一个 state 设置函数 ( `[value, setValue]`)                 |
| 更改时不会触发重新渲染                                          | 更改时触发重新渲染。                                                                                    |
| 可变 —— 你可以在渲染过程之外修改和更新 `current` 的值。                  | “不可变” —— 你必须使用 state 设置函数来修改 state 变量，从而排队重新渲染。                                               |
| 你不应在渲染期间读取（或写入） `current` 值。                         | 你可以随时读取 state。但是，每次渲染都有自己不变的 state [快照](https://zh-hans.react.dev/learn/state-as-a-snapshot)。 |
### 何时使用 ref？
通常，当你的组件需要“跳出” React 并与外部 API 通信时，你会用到 ref —— 通常是不会影响组件外观的浏览器 API。以下是这些罕见情况中的几个：
- 存储 [timeout ID](https://developer.mozilla.org/docs/Web/API/setTimeout)
- 存储和操作 [DOM 元素](https://developer.mozilla.org/docs/Web/API/Element)
- 存储不需要被用来计算 JSX 的其他对象。
如果你的组件需要存储一些值，但不影响渲染逻辑，请选择 ref。
### ref 的最佳实践
遵循这些原则将使你的组件更具可预测性：
- **将 ref 视为脱围机制**。当你使用外部系统或浏览器 API 时，ref 很有用。如果你很大一部分应用程序逻辑和数据流都依赖于 ref，你可能需要重新考虑你的方法。
- **不要在渲染过程中读取或写入 `ref.current`。** 如果渲染过程中需要某些信息，请使用 [state](https://zh-hans.react.dev/learn/state-a-components-memory) 代替。由于 React 不知道 `ref.current` 何时发生变化，即使在渲染时读取它也会使组件的行为难以预测。（唯一的例外是像 `if (!ref.current) ref.current = new Thing()` 这样的代码，它只在第一次渲染期间设置一次 ref。）
## 使用 ref 操纵 DOM
### 获取指向节点的 ref
要访问由 React 管理的 DOM 节点，首先，引入 `useRef` Hook：
```jsx
import { useRef } from 'react';
```

然后，在你的组件中使用它声明一个 ref：
```jsx
const myRef = useRef(null);
```

最后，将 ref 作为 `ref` 属性值传递给想要获取的 DOM 节点的 JSX 标签：
```jsx
<div ref={myRef}>
```

`useRef` Hook 返回一个对象，该对象有一个名为 `current` 的属性。最初，`myRef.current` 是 `null`。当 React 为这个 `<div>` 创建一个 DOM 节点时，React 会把对该节点的引用放入 `myRef.current`。然后，你可以从 [事件处理器](https://zh-hans.react.dev/learn/responding-to-events) 访问此 DOM 节点，并使用在其上定义的内置[浏览器 API](https://developer.mozilla.org/docs/Web/API/Element)。
```jsx
// 你可以使用任意浏览器 API，例如：
myRef.current.scrollIntoView();
```
### 案例：使文本输入框获得焦点
```jsx
import { useRef } from 'react';

export default function Form() {
  const inputRef = useRef(null);

  function handleClick() {
    inputRef.current.focus();
  }

  return (
    <>
      <input ref={inputRef} />
      <button onClick={handleClick}>
        聚焦输入框
      </button>
    </>
  );
}
```