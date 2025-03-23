通常来说，你会通过 props 将信息从父组件传递到子组件。但是，如果你必须通过许多中间组件向下传递 props，或是在你应用中的许多组件需要相同的信息，传递 props 会变的十分冗长和不便。**Context** 允许父组件向其下层无论多深的任何组件提供信息，而无需通过 props 显式传递。

传递 props 是将数据通过 UI 树显式传递到使用它的组件的好方法。但是当你需要在组件树中深层传递参数以及需要在组件间复用相同的参数时，传递 props 就会变得很麻烦。最近的根节点父组件可能离需要数据的组件很远，状态提升 到太高的层级会导致 “逐层传递 props” 的情况。
## Context：传递 Props 的另一种方法
Context 让父组件可以为它下面的整个组件树提供数据。
```jsx
<Section>
  <Heading level={3}>关于</Heading>
  <Heading level={3}>照片</Heading>
  <Heading level={3}>视频</Heading>
</Section>
```
将 `level` 参数传递给 `<Section>` 组件而不是传给 `<Heading>` 组件看起来更好一些。这样的话你可以强制使同一个 section 中的所有标题都有相同的尺寸：
```jsx
<Section level={3}>
  <Heading>关于</Heading>
  <Heading>照片</Heading>
  <Heading>视频</Heading>
</Section>
```
但是 `<Heading>` 组件是如何知道离它最近的 `<Section>` 的 level 的呢？**这需要子组件可以通过某种方式“访问”到组件树中某处在其上层的数据。**
### Step 1：创建 Context
首先，你需要创建这个 context，并 **将其从一个文件中导出**，这样你的组件才可以使用它：
```jsx
import { createContext } from 'react';

export const LevelContext = createContext(1);
```
`createContext` 只需**默认值**这么一个参数。在这里, `1` 表示最大的标题级别，但是你可以传递任何类型的值（甚至可以传入一个对象）。你将在下一个步骤中见识到默认值的意义。
### Step2：使用 Context
```jsx
import { useContext } from 'react';
import { LevelContext } from './LevelContext.js';

export default function Heading({ children }) {
  const level = useContext(LevelContext);
  // ...
}
```
### Step3：提供 Context
`Section` 组件目前渲染传入它的子组件：
```jsx
export default function Section({ children }) {
  return (
    <section className="section">
      {children}
    </section>
  );
}
```
**把它们用 context provider 包裹起来** 以提供 `LevelContext` 给它们：
```jsx
import { LevelContext } from './LevelContext.js';

export default function Section({ level, children }) {
  return (
    <section className="section">
      <LevelContext value={level}>
        {children}
      </LevelContext>
    </section>
  );
}
```
这告诉 React：“如果在 `<Section>` 组件中的任何子组件请求 `LevelContext`，给他们这个 `level`。”组件会使用 UI 树中在它上层最近的那个 `<LevelContext>` 传递过来的值。