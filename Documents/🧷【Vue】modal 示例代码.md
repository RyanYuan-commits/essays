## 1 子组件

```vue
<template>  
  <div>  
    <a-modal  
      v-model:visible="visible"  
      title="JSON预览"  
      :width="800"  
      :footer="null"  
      @cancel="handleCancel"  
    >  
      <div class="json-preview-container">  
        <div class="json-actions">  
          <a-button type="primary" size="small" @click="copyJson">  
            <template #icon><CopyOutlined /></template>  
            复制  
          </a-button>  
        </div>  
        <div class="json-content">  
          <pre>{{ formattedJson }}</pre>  
        </div>  
      </div>  
    </a-modal>  
  </div>  
</template>  
  
<script setup lang="ts">  
import { ref, computed, watch } from 'vue';  
import { message } from 'ant-design-vue';  
import { CopyOutlined } from '@ant-design/icons-vue';  
  
const props = defineProps({  
  jsonObject: {  
    type: Object,  
    required: true  
  },  
  showModal: {  
    type: Boolean,  
    required: true  
  }  
});  
  
// 定义emit事件，用于通知父组件弹窗状态变化  
const emit = defineEmits(['update:showModal']);  
  
const visible = ref(props.showModal);  
  
// 监听父组件传递的showModal变化  
watch(() => props.showModal, (newVal) => {  
  visible.value = newVal;  
});  
  
// 计算格式化后的JSON  
const formattedJson = computed(() => {  
  try {  
    return JSON.stringify(props.jsonObject, null, 2);  
  } catch (error) {  
    return "无效的JSON数据";  
  }  
});  
  
// 关闭弹窗（子组件内部控制）  
const handleCancel = () => {  
  visible.value = false;  
  emit('update:showModal', false);  
};  
  
// 复制JSON  
const copyJson = async () => {  
  try {  
    await navigator.clipboard.writeText(formattedJson.value);  
    message.success('JSON已复制到剪贴板');  
  } catch (err) {  
    message.error('复制失败，请手动复制');  
    console.error('复制失败:', err);  
  }  
};  
</script>  
  
<style scoped>  
.json-preview-container {  
  display: flex;  
  flex-direction: column;  
  gap: 12px;  
}  
.json-actions {  
  display: flex;  
  gap: 8px;  
  justify-content: flex-end;  
}  
.json-content {  
  background-color: #f5f5f5;  
  border: 1px solid #d9d9d9;  
  border-radius: 4px;  
  padding: 12px;  
  max-height: 400px;  
  overflow: auto;  
}  
.json-content pre {  
  margin: 0;  
  white-space: pre-wrap;  
  word-break: break-all;  
  font-family: 'Consolas', 'Monaco', 'Courier New', monospace;  
  font-size: 14px;  
  line-height: 1.5;  
}  
</style>
```

## 2 父组件

```vue
<template>
  <div>
    <a-button @click="showModal = true">显示JSON预览</a-button>
    
    <json-preview-modal
      v-model:show-modal="showModal"
      :json-object="jsonData"
    />
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue';
import JsonPreviewModal from './JsonPreviewModal.vue';

const showModal = ref(false);
const jsonData = ref({
  name: '示例数据',
  value: 123,
  items: [1, 2, 3]
});
</script>
```