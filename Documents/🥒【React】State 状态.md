## State: 组件的记忆
组件通常需要根据交互更改屏幕上显示的内容。输入表单应该更新输入字段，单击轮播图上的“下一个”应该更改显示的图片，单击“购买”应该将商品放入购物车。组件需要“记住”某些东西：当前输入值、当前图片、购物车。在 React 中，这种组件特有的记忆被称为 **state**。
### 添加一个 State 变量
要添加 state 变量，先从文件顶部的 React 中导入 `useState`：
```jsx
import { useState } from 'react';
const [index, setIndex] = useState(0);
```
当你调用 [`useState`](https://zh-hans.react.dev/reference/react/useState) 时，你是在告诉 React 你想让这个组件记住一些东西；
例如，在上面的案例中，你希望 React 记住 index，`useState` 的唯一参数是 state 变量的**初始值**。在这个例子中，`index` 的初始值被`useState(0)`设置为 `0`。
每次你的组件渲染时，`useState` 都会给你一个包含两个值的数组：
1. **state 变量** (`index`) 会保存上次渲染的值。
2. **state setter 函数** (`setIndex`) 可以更新 state 变量并触发 React 重新渲染组件。

以下是实际发生的情况：
1. **组件进行第一次渲染。** 因为你将 `0` 作为 `index` 的初始值传递给 `useState`，它将返回 `[0, setIndex]`。 React 记住 `0` 是最新的 state 值。
2. **你更新了 state**。当用户点击按钮时，它会调用 `setIndex(index + 1)`。 `index` 是 `0`，所以它是 `setIndex(1)`。这告诉 React 现在记住 `index` 是 `1` 并触发下一次渲染。
3. **组件进行第二次渲染**。React 仍然看到 `useState(0)`，但是因为 React _记住_ 了你将 `index` 设置为了 `1`，它将返回 `[1, setIndex]`。
4. 以此类推！

## State 的快照特性
### State 如同一张快照
下面的案例中，点击一次按钮，得到的会是 1 而不是 3。
```jsx
import { useState } from 'react';

export default function Counter() {
  const [number, setNumber] = useState(0);

  return (
    <>
      <h1>{number}</h1>
      <button onClick={() => {
        setNumber(number + 1);
        setNumber(number + 1);
        setNumber(number + 1);
      }}>+3</button>
    </>
  )
}
```
这是因为 **设置 state 只会为下一次渲染变更 state 的值**，我们可以理解为 React 将这些更新操作放到了一个队列中，等待下次渲染的时候，从队列中取出这些更新操作然后再次执行，例如，在 number 为 0 的时候，提交的三条更新语句是这样的：
```jsx
setNumber(0 + 1);
setNumber(0 + 1);
setNumber(0 + 1);
```
React 会连续三次将 number 赋值为 1。
### 将一系列更新加入队列
那只要将队列中的更新操作改为：使 number 递增 1，就可以完成上面的逻辑了；具体的方法就是向 set 方法传递值的时候，传递的是一个更新函数；例如上面的案例，最终得到的就是这样的效果：
```jsx
setNumber(n => n + 1);
setNumber(n => n + 1);
setNumber(n => n + 1);
```
面是 React 在执行事件处理函数时处理这几行代码的过程：
1. `setNumber(n => n + 1)`：`n => n + 1` 是一个函数。React 将它加入队列。
2. `setNumber(n => n + 1)`：`n => n + 1` 是一个函数。React 将它加入队列。
3. `setNumber(n => n + 1)`：`n => n + 1` 是一个函数。React 将它加入队列。

在替换 State 之后更新 State 会发生什么？
```jsx
setNumber(number + 5);
setNumber(n => n + 1);
```
假如 number 为 0，最终得到就就是 6。

在更新 state 后替换 state 会发生什么？
```jsx
setNumber(number + 5);
setNumber(n => n + 1);
setNumber(42);
```
假设 number 为 0，最终得到的结果是 42，更新队列是这样的：
- 将 number 设置为 5
- 给 number 累加 1
- 将 number 设置为 42

## 更新 state 中的对象
### 什么是 mutation？
现在考虑 state 中存放对象的情况：
```jsx
const [position, setPosition] = useState({ x: 0, y: 0 });
```
从技术上来讲，可以改变对象自身的内容。**当你这样做时，就制造了一个 mutation**：
```jsx
position.x = 5;
```
然而，虽然严格来说 React state 中存放的对象是可变的，但你应该像处理数字、布尔值、字符串一样将它们视为不可变的。因此你应该替换它们的值，而不是对它们进行修改。

应该 **把所有存放在 state 中的 JavaScript 对象都视为只读的**。
### 如何优雅的更新对象？
#### 使用展开语法复制对象
在只需要修改少数的属性的情况下，可以使用展开语法来优雅的复制对象：
```jsx
setPerson({
  firstName: e.target.value, // 从 input 中获取新的 first name
  lastName: person.lastName,
  email: person.email
});

setPerson({
  ...person, // 复制上一个 person 中的所有字段
  firstName: e.target.value // 但是覆盖 firstName 字段 
});
```
请注意 `...` 展开语法本质是是“浅拷贝”——它只会复制一层。这使得它的执行速度很快，但是也意味着当你想要更新一个嵌套属性时，你必须得多次使用展开语法。
#### 更新一个嵌套对象
例如，这种结构的嵌套对象，如果你想要更新 `person.artwork.city` 的值，应该如何操作？
```jsx
const [person, setPerson] = useState({
  name: 'Niki de Saint Phalle',
  artwork: {
    title: 'Blue Nana',
    city: 'Hamburg',
    image: 'https://i.imgur.com/Sd1AgUOm.jpg',
  }
});
```

为了修改 `city` 的值，你首先需要创建一个新的 `artwork` 对象（其中预先填充了上一个 `artwork` 对象中的数据），然后创建一个新的 `person` 对象，并使得其中的 `artwork` 属性指向新创建的 `artwork` 对象：
```jsx
const nextArtwork = { ...person.artwork, city: 'New Delhi' };
const nextPerson = { ...person, artwork: nextArtwork };
setPerson(nextPerson);
```
或者，写成一个函数调用：
```jsx
setPerson({
  ...person, // 复制其它字段的数据 
  artwork: { // 替换 artwork 字段 
    ...person.artwork, // 复制之前 person.artwork 中的数据
    city: 'New Delhi' // 但是将 city 的值替换为 New Delhi！
  }
});
```
## 更新 State 中的数组
数组是另外一种可以存储在 state 中的 JavaScript 对象，它虽然是可变的，但是却应该被视为不可变。同对象一样，当你想要更新存储于 state 中的数组时，你需要创建一个新的数组（或者创建一份已有数组的拷贝值），并使用新数组设置 state。
### 向数组中添加元素
`push()` 会直接修改原始数组，而你不希望这样，相反，你应该创建一个 **新** 数组，其包含了原始数组的所有元素 **以及** 一个在末尾的新元素。这可以通过很多种方法实现，最简单的一种就是使用 `...` 数组展开 语法：
```jsx
setArtists( // 替换 state
  [ // 是通过传入一个新数组实现的
    ...artists, // 新数组包含原数组的所有元素
    { id: nextId++, name: name } // 并在末尾添加了一个新的元素
  ]
);
```
### 从数组中删除元素
从数组中删除一个元素最简单的方法就是将它**过滤出去**。换句话说，你需要生成一个不包含该元素的新数组。这可以通过 `filter` 方法实现，例如：
```jsx
import { useState } from 'react';

let initialArtists = [
  { id: 0, name: 'Marta Colvin Andrade' },
  { id: 1, name: 'Lamidi Olonade Fakeye'},
  { id: 2, name: 'Louise Nevelson'},
];

export default function List() {
  const [artists, setArtists] = useState(
    initialArtists
  );

  return (
    <>
      <h1>振奋人心的雕塑家们：</h1>
      <ul>
        {artists.map(artist => (
          <li key={artist.id}>
            {artist.name}{' '}
            <button onClick={() => {
              setArtists(
                artists.filter(a =>
                  a.id !== artist.id
                )
              );
            }}>
              删除
            </button>
          </li>
        ))}
      </ul>
    </>
  );
}
```
### 转换数组
如果你想改变数组中的某些或全部元素，你可以用 `map()` 创建一个**新**数组。你传入 `map` 的函数决定了要根据每个元素的值或索引（或二者都要）对元素做何处理。
在下面的例子中，一个数组记录了两个圆形和一个正方形的坐标。当你点击按钮时，仅有两个圆形会向下移动 100 像素。这是通过使用 `map()` 生成一个新数组实现的。
```jsx
  function handleClick() {
    const nextShapes = shapes.map(shape => {
      if (shape.type === 'square') {
        // 不作改变
        return shape;
      } else {
        // 返回一个新的圆形，位置在下方 50px 处
        return {
          ...shape,
          y: shape.y + 50,
        };
      }
    });
    // 使用新的数组进行重渲染
    setShapes(nextShapes);
  }
```
### 替换数组中的元素
想要替换数组中一个或多个元素是非常常见的。类似 `arr[0] = 'bird'` 这样的赋值语句会直接修改原始数组，所以在这种情况下，你也应该使用 `map`。
```jsx
function handleIncrementClick(index) {
	const nextCounters = counters.map((c, i) => {
	  if (i === index) {
		// 递增被点击的计数器数值
		return c + 1;
	  } else {
		// 其余部分不发生变化
		return c;
	  }
	});
	setCounters(nextCounters);
}
```
### 向数组中插入元素
有时，你也许想向数组特定位置插入一个元素，这个位置既不在数组开头，也不在末尾。为此，你可以将数组展开运算符 `...` 和 `slice()` 方法一起使用。`slice()` 方法让你从数组中切出“一片”。为了将元素插入数组，你需要先展开原数组在插入点之前的切片，然后插入新元素，最后展开原数组中剩下的部分。
```jsx
function handleClick() {
	const insertAt = 1; // 可能是任何索引
	const nextArtists = [
		// 插入点之前的元素：
		...artists.slice(0, insertAt),
		// 新的元素：
		{ id: nextId++, name: name },
		// 插入点之后的元素：
		...artists.slice(insertAt)
	];
	setArtists(nextArtists);
	setName('');
}
```
### 其他改变数组的情况
总会有一些事，是你仅仅依靠展开运算符和 `map()` 或者 `filter()` 等不会直接修改原值的方法所无法做到的。例如，你可能想翻转数组，或是对数组排序。而 JavaScript 中的 `reverse()` 和 `sort()` 方法会改变原数组，所以你无法直接使用它们。

**然而，你可以先拷贝这个数组，再改变这个拷贝后的值。**

```jsx
export default function List() {
	const [list, setList] = useState(initialList);
	
	function handleClick() {
	const nextList = [...list];
	nextList.reverse();
	setList(nextList);
}
```