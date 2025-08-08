组件是 React 的核心概念之一，它们是构建用户界面 UI 的基础。

## 定义组件
一直以来，创建网页时，Web 开发人员会用标签描述内容，然后通过 JavaScript 来增加交互。这种在 Web 上添加交互的方式能产生出色的效果。现在许多网站和全部应用都需要交互。

React 最为重视交互性且使用了相同的处理方式：**React 组件是一段可以 使用标签进行扩展 的 JavaScript 函数**。如下所示（你可以编辑下面的示例）：
```js
export default function Profile() {
  return (
    <img
      src="https://i.imgur.com/MK3eW3Am.jpg"
      alt="Katherine Johnson"
    />
  )
}
```
以下是构建组件的方法：
### 第一步：导出组件
`export default` 前缀是一种 [JavaScript 标准语法](https://developer.mozilla.org/docs/web/javascript/reference/statements/export)（非 React 的特性）。它允许你导出一个文件中的主要函数以便你以后可以从其他文件引入它。

### 第二步：定义函数
使用 `function Profile() { }` 定义名为 `Profile` 的 JavaScript 函数。
React 组件是常规的 JavaScript 函数，但 **组件的名称必须以大写字母开头**，否则它们将无法运行！

### 第三步：添加标签
返回语句可以全写在一行上，如下面组件中所示：
```js
return <img src="https://i.imgur.com/MK3eW3As.jpg" alt="Katherine Johnson" />;
```

但是如果你的标签和 return 关键字不在同一行，则必须把它包裹在一对括号中；
	如果没有括号包裹的话，任何在 `return` 下一行的代码都将被忽略。
```js
return (
  <div>
    <img src="https://i.imgur.com/MK3eW3As.jpg" alt="Katherine Johnson" />
  </div>
);
```

## 使用组件
现在已经定义了 Profile 组件，你可以在其他组件中使用它，例如，你可以导出一个内部使用了多个 Profile 组件的 Gallery 组件。
```js
function Profile() {
  return (
    <img
      src="https://i.imgur.com/MK3eW3As.jpg"
      alt="Katherine Johnson"
    />
  );
}

export default function Gallery() {
  return (
    <section>
      <h1>了不起的科学家</h1>
      <Profile />
      <Profile />
      <Profile />
    </section>
  );
}
```
### 浏览器所看到的
注意下面两者的区别：
- `<section>` 是小写的，所以 React 知道我们指的是 HTML 标签。
- `<Profile />` 以大写 `P` 开头，所以 React 知道我们想要使用名为 `Profile` 的组件。
然而 `Profile` 包含更多的 HTML：`<img />`。这是浏览器最后所看到的：
```html
<section>
  <h1>了不起的科学家</h1>
  <img src="https://i.imgur.com/MK3eW3As.jpg" alt="Katherine Johnson" />
  <img src="https://i.imgur.com/MK3eW3As.jpg" alt="Katherine Johnson" />
  <img src="https://i.imgur.com/MK3eW3As.jpg" alt="Katherine Johnson" />
</section>
```
### 嵌套和组织组件
组件是常规的 JavaScript 函数，所以你可以将多个组件保存在同一份文件中，当组件相对较小且彼此尽敏关联的时候，这是一种相对省事的方式，但是当这个文件变得臃肿，你也可以随时将 Profile 移动到单独的文件夹中。
组件可以渲染其他组件，但是请不要嵌套他们的定义：
```js
export default function Gallery() {
  // 🔴 永远不要在组件中定义组件
  function Profile() {
    // ...
  }
  // ...
}
```
上面的代码非常慢，并且会导致 Bug 的产生，因此，你应该在顶层去定义每个组件：
```js
export default function Gallery() {
  // ...
}

// ✅ 在顶层声明组件
function Profile() {
  // ...
}
```
