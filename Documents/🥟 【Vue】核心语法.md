![[vue 简介.png|300]]
官方地址：[https://cn.vuejs.org/](https://cn.vuejs.org/)
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    <div id="app">
        <h2>第一部分：响应式数据</h1>
        <h3>{{ title }}</h1>
        <p>{{ content }}</p>

        <h2>第二部分：方法与计算属性</h1>
        <p>{{ output() }}</p>
        <p>{{ outputContent }}</p>
        <p>{{ outputContent }}</p>
        <p>{{ outputContent }}</p>

        <h3>第三部分: 侦听器与指令</h3>
        <h4>内容指令</h4>
        <p v-text="textContent" />
        <p v-html="htmlContent" />

        <h4>渲染指令</h4>
        <p v-if="bool">v-if: 彻底销毁</p>
        <p v-show="bool">v-show: display: none</p>

        <h4>属性指令</h4>
        <!-- 设置鼠标悬浮内容 -->
        <p v-bind:title="title">这是内容</p>
        <!-- 语法糖 -->
        <p :title="title">这是内容</p>

        <h4>事件指令</h4>
        <button v-on:click="output()">按钮</button>

        <h4>表单指令</h4>
        <input type="text" v-model="title" />

        <h3>第四部分：修饰符</h4>
        <input type="text" v-model.trim="title">

    </div>
    <script src="https://cdn.jsdelivr.net/npm/vue@2/dist/vue.js"></script>
    <script>
        const vm = new Vue({
            el: '#app',
            data() {
                return {
                    title: "这是标题",
                    content: "这是文本",
                    textContent: "v-text",
                    htmlContent: `<div style="color: red">v-html</div>`,
                    bool: true
                }
            },
            // 方法
            methods: {
                output() {
                    console.log('output 执行');
                    return '标题为: ' + this.title + ', ' + '内容为: ' + this.content;
                }
            },
            // 计算属性，缓存机制，观察控制台，只会输出一次「output 执行了」
            computed: {
                outputContent() {
                    console.log('computed: 计算')
                    return '标题为: ' + this.title + ', ' + '内容为: ' + this.content;
                }
            },
            // 侦听器：监测响应式数据的变化
            watch: {
                title(newVal, oldVal) {
                    console.log(newVal, oldVal);
                }
            }
        })
    </script>
</body>
</html>
```