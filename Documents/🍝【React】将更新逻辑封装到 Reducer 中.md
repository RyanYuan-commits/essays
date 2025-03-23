## 使用 reducer 整合状态逻辑
随着组件复杂度的增加，你将很难一眼看清所有的组件状态更新逻辑。例如，下面的 `TaskApp` 组件有一个数组类型的状态 `tasks`，并通过三个不同的事件处理程序来实现任务的添加、删除和修改：
```jsx
import { useState } from 'react';
import AddTask from './AddTask.js';
import TaskList from './TaskList.js';

export default function TaskApp() {
  const [tasks, setTasks] = useState(initialTasks);

  function handleAddTask(text) {
    setTasks([
      ...tasks,
      {
        id: nextId++,
        text: text,
        done: false,
      },
    ]);
  }

  function handleChangeTask(task) {
    setTasks(
      tasks.map((t) => {
        if (t.id === task.id) {
          return task;
        } else {
          return t;
        }
      })
    );
  }

  function handleDeleteTask(taskId) {
    setTasks(tasks.filter((t) => t.id !== taskId));
  }

  return (
    <>
      <h1>布拉格的行程安排</h1>
      <AddTask onAddTask={handleAddTask} />
      <TaskList
        tasks={tasks}
        onChangeTask={handleChangeTask}
        onDeleteTask={handleDeleteTask}
      />
    </>
  );
}

let nextId = 3;
const initialTasks = [
  {id: 0, text: '参观卡夫卡博物馆', done: true},
  {id: 1, text: '看木偶戏', done: false},
  {id: 2, text: '打卡列侬墙', done: false},
];
```
这个组件的每个事件处理程序都通过 `setTasks` 来更新状态。随着这个组件的不断迭代，其状态逻辑也会越来越多。为了降低这种复杂度，并让所有逻辑都可以存放在一个易于理解的地方，你可以将这些状态逻辑移到组件之外的一个称为 **reducer** 的函数中。

Reducer 是处理状态的另一种方式。你可以通过三个步骤将 `useState` 迁移到 `useReducer`：
1. 将设置状态的逻辑 **修改** 成 dispatch 的一个 action；
2. **编写** 一个 reducer 函数；
3. 在你的组件中 **使用** reducer。
### 第 1 步: 将设置状态的逻辑修改成 dispatch 的一个 action
```jsx
function handleAddTask(text) {
  dispatch({
    type: 'added',
    id: nextId++,
    text: text,
  });
}

function handleChangeTask(task) {
  dispatch({
    type: 'changed',
    task: task,
  });
}

function handleDeleteTask(taskId) {
  dispatch({
    type: 'deleted',
    id: taskId,
  });
}
```
传递给 `dispatch` 的对象叫做 “action”：
```jsx
function handleDeleteTask(taskId) {
  dispatch(
    // "action" 对象：
    {
      type: 'deleted',
      id: taskId,
    }
  );
}
```
它是一个普通的 JavaScript 对象。它的结构是由你决定的，但通常来说，它应该至少包含可以表明 **发生了什么事情** 的信息。（在后面的步骤中，你将会学习如何添加一个 `dispatch` 函数。）
### 第 2 步：编写一个 reducer 函数
reducer 函数就是你放置状态逻辑的地方。它接受两个参数，分别为当前 state 和 action 对象，并且返回的是更新后的 state：
```jsx
function yourReducer(state, action) {
  // 给 React 返回更新后的状态
}
```
React 会将状态设置为你从 reducer 返回的状态。
在这个例子中，要将状态设置逻辑从事件处理程序移到 reducer 函数中，你需要：
1. 声明当前状态（`tasks`）作为第一个参数；
2. 声明 `action` 对象作为第二个参数；
3. 从 `reducer` 返回 **下一个** 状态（React 会将旧的状态设置为这个最新的状态）。
下面是所有迁移到 `reducer` 函数的状态设置逻辑：
```jsx
function tasksReducer(tasks, action) {
  if (action.type === 'added') {
    return [
      ...tasks,
      {
        id: action.id,
        text: action.text,
        done: false,
      },
    ];
  } else if (action.type === 'changed') {
    return tasks.map((t) => {
      if (t.id === action.task.id) {
        return action.task;
      } else {
        return t;
      }
    });
  } else if (action.type === 'deleted') {
    return tasks.filter((t) => t.id !== action.id);
  } else {
    throw Error('未知 action: ' + action.type);
  }
}
```
### 第 3 步: 在组件中使用 reducer
```jsx
import { useReducer } from 'react';
const [tasks, setTasks] = useState(initialTasks);
const [tasks, dispatch] = useReducer(tasksReducer, initialTasks);
```
`useReducer` 钩子接受 2 个参数：
1. 一个 reducer 函数
2. 一个初始的 state
它返回如下内容：
3. 一个有状态的值
4. 一个 dispatch 函数（用来 “派发” 用户操作给 reducer）