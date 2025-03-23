使用 React 可以在 JSX 中添加 **事件处理函数**。其中事件处理函数为自定义函数，它将在响应交互（如点击、悬停、表单输入框获得焦点等）时触发。
## 事件处理
### 添加事件处理函数
如需添加一个事件处理函数，你需要先定义一个函数，然后 **将其作为 prop 传入** 合适的 JSX 标签。例如，这里有一个没绑定任何事件的按钮：
1. 在 `Button` 组件 **内部** 声明一个名为 `handleClick` 的函数。
2. 实现函数内部的逻辑（使用 `alert` 来显示消息）。
3. 添加 `onClick={handleClick}` 到 `<button>` JSX 中。
```js
export default function Button() {
  function handleClick() {
    alert('你点击了我！');
  }

  return (
    <button onClick={handleClick}>
      点我
    </button>
  );
}
```
其他更简洁的写法：
```js
<button onClick={function handleClick() {
  alert('你点击了我！');
}}>
<button onClick={() => {
  alert('你点击了我！');
}}>
```
### 在事件处理函数中访问 Props
事件处理函数是声明在组件内部的，因此它可以直接访问组件的 Props。
```js
function AlertButton({ message, children }) {
  return (
    <button onClick={() => alert(message)}>
      {children}
    </button>
  );
}

export default function Toolbar() {
  return (
    <div>
      <AlertButton message="正在播放！">
        播放电影
      </AlertButton>
      <AlertButton message="正在上传！">
        上传图片
      </AlertButton>
    </div>
  );
}
```
## 事件传播
### 什么是事件传播？
事件处理函数还将捕获任何来自子组件的事件。通常，我们会说事件会沿着树向上“冒泡”或“传播”：它从事件发生的地方开始，然后沿着树向上传播。
```jsx
export default function Toolbar() {
  return (
    <div className="Toolbar" onClick={() => {
      alert('你点击了 toolbar ！');
    }}>
      <button onClick={() => alert('正在播放！')}>
        播放电影
      </button>
      <button onClick={() => alert('正在上传！')}>
        上传图片
      </button>
    </div>
  );
}
```
如果你点击任一按钮，它自身的 `onClick` 将首先执行，然后父级 `<div>` 的 `onClick` 会接着执行。因此会出现两条消息。如果你点击 toolbar 本身，将只有父级 `<div>` 的 `onClick` 会执行。
在 React 中所有事件都会传播，除了 `onScroll`，它仅适用于你附加到的 JSX 标签。
### 阻止传播
事件处理函数接受一个事件对象作为唯一参数，按照惯例，它通常被称为 `e` ，代表 “event”（事件）。你可以使用此对象来读取有关事件的信息。
这个事件对象还允许你阻止传播。如果你想阻止一个事件到达父组件，你需要像下面 `Button` 组件那样调用 `e.stopPropagation()` ：
```jsx
function Button({ onClick, children }) {
  return (
    <button onClick={e => {
      e.stopPropagation();
      onClick();
    }}>
      {children}
    </button>
  );
}

export default function Toolbar() {
  return (
    <div className="Toolbar" onClick={() => {
      alert('你点击了 toolbar ！');
    }}>
      <Button onClick={() => alert('正在播放！')}>
        播放电影
      </Button>
      <Button onClick={() => alert('正在上传！')}>
        上传图片
      </Button>
    </div>
  );
}
```
## 阻止默认行为
某些浏览器事件具有和事件相关联的默认行为。例如，点击 form 表单内部的按钮会触发表单提交事件，并且将重新加载整个页面。
```js
export default function Signup() {
  return (
    <form onSubmit={() => alert('提交表单！')}>
      <input />
      <button>发送</button>
    </form>
  );
}
```
你可以调用事件对象中的 `e.preventDefault()` 来阻止这种情况发生：
```jsx
export default function Signup() {
  return (
    <form onSubmit={e => {
      e.preventDefault();
      alert('提交表单！');
    }}>
      <input />
      <button>发送</button>
    </form>
  );
}
```