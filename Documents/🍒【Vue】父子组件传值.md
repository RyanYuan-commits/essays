在 Vue 3 中，父子组件之间的传值可以通过 `props` 和 `emits` 来实现。以下是一个简单的示例，展示如何在 Vue 3 中实现父子组件之间的通信。
>[!info] 父组件
```vue
<template>
  <div>
    <h1>父组件</h1>
    <!-- 向子组件传递一个 message 属性 -->
    <ChildComponent :message="parentMessage" @update-message="handleUpdateMessage" />
    <p>从子组件接收到的消息: {{ receivedMessage }}</p>
  </div>
</template>

<script>
import { ref } from 'vue';
import ChildComponent from './ChildComponent.vue';
export default {
  components: {
    ChildComponent
  },
  setup() {
    const parentMessage = ref('Hello from Parent');
    const receivedMessage = ref('');

    const handleUpdateMessage = (newMessage) => {
      receivedMessage.value = newMessage;
    };

    return {
      parentMessage,
      receivedMessage,
      handleUpdateMessage
    };
  }
};
</script>
```


>[!info] 子组件
```vue
<template>
  <div>
    <h2>子组件</h2>
    <p>从父组件接收到的消息: {{ message }}</p>
    <button @click="sendMessageToParent">发送消息给父组件</button>
  </div>
</template>

<script>
import { defineComponent } from 'vue';
export default defineComponent({
  props: {
    message: {
      type: String,
      required: true
    }
  },
  emits: ['update-message'],
  setup(props, { emit }) {
    const sendMessageToParent = () => {
      emit('update-message', 'Hello from Child');
    };
    return {
      sendMessageToParent
    };
  }
});
</script>
```
### 说明
1. **父组件**：通过 `props` 向子组件传递数据，并通过 `emits` 接收子组件发送的事件。
2. **子组件**：接收父组件传递的 `props`，并通过 `emit` 方法向父组件发送事件。