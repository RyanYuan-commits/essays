## 导入和导出一个组件
1. **创建** 一个新的 JS 文件来存放该组件。
2. **导出** 该文件中的函数组件
3. 在需要使用该组件的文件中 **导入**
引入过程中，你可能会遇到一些文件并未添加 `.js` 文件后缀，如下所示：
```js
import Gallery from './Gallery';
```
无论是 './Gallery.js' 还是 './Gallery'，在 React 里都能正常使用，只是前者更符合 原生 ES 模块。

## 在同一个文件中导入和导出多个组件
如果你只想展示一个 `Profile` 组，而不展示整个图集。你也可以导出 `Profile` 组件。但 `Gallery.js` 中已包含 **默认** 导出，此时，你不能定义 **两个** 默认导出。但你可以将其在新文件中进行默认导出，或者将 `Profile` 进行 **具名** 导出。**同一文件中，有且仅有一个默认导出，但可以有多个具名导出！**

首先，用具名导出的方式，将 `Profile` 组件从 `Gallery.js` **导出**（不使用 `default` 关键字）：
```js
export function Profile() {
  // ...
}
```

接着，用具名导入的方式，从 `Gallery.js` 文件中 **导入** `Profile` 组件（用大括号）:
```js
export default function App() {
  return <Profile />;
}
```

现在，`Gallery.js` 包含两个导出：一个是默认导出的 `Gallery`，另一个是具名导出的 `Profile`。`App.js` 中均导入了这两个组件。尝试将 `<Profile />` 改成 `<Gallery />`，回到示例中：
```js
import Gallery from './Gallery.js';
import { Profile } from './Gallery.js';

export default function App() {
  return (
    <Profile />
  );
}
```
