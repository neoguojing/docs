# vue3 && typescript

## 常用库
- lodash ： 防抖和节流
## vue3

### 基础概念
- 标签/元素：div 和span等
- 属性/attribute ： div或span等尖括号内的内容：如 <span v-bind:title="message">，title即为一个属性
### vue特性
- createApp(App).use(router).mount('#app'): 创建一个应用，指定根组件（App.vue）,使用路由，并和#app绑定
- 在index.html中：<div id="app"></div> ：指定加载的vue应用
- App.vue：根组件，DOM渲染的起点，任何相关的组件需要被展示，就需要和App.vue建立联系
 
 
#### 指令
- v-bind：将标签属性值与组件值绑定；改变组件值，可以影响DOM树，从而影响到展示；
- v-bind:href="url"：将标签的href属性和url的值绑定
- v-on:添加一个事件监听器；v-on:click=;<button @click="one($event), two($event)"> 多个事件监听；
 ```
 <a @click.once="doThis"></a>
 <form @submit.prevent="onSubmit"></form>
 ```
- v-model：实现表单输入（input）和应用状态之间的双向绑定;v-model.lazy 在change事件时触发，而非input
 ```
 text 和 textarea 元素使用 value property 和 input 事件；
checkbox 和 radio 使用 checked property 和 change 事件；
select 字段将 value 作为 prop 并将 change 作为事件。
 ```
- v-if/v-for：
 ```
 <ul id="array-rendering">
  <li v-for="item in items">
    {{ item.message }}
  </li>
</ul>
 ```
- v-once：<span v-once>这个将不会改变: {{ msg }}</span>
- v-show：简单的切换css的display ，不能在template元素中使用
#### 模板
- .vue文件中的html代码
- 文本插值： <span>Message: {{ msg }}</span>
- 插槽：使用<slot>占位
#### 组件
- 根组件和其他组件没什么不同
- 每个组件实例都有生命周期钩子函数：  created()，mounted等，可以设置监控等
- 组件创建：
- 全局组件组测：app.component ，所以组件可以相互引用
- 局部组件注册：
 ```
<!--  定义组件 -->
 import { Options, Vue } from 'vue-class-component'
 export default class HelloWorld extends Vue {}
<!-- 被引用 -->
 import { Options, Vue } from 'vue-class-component'
import HelloWorld from '@/components/HelloWorld.vue' //导入
 @Options({ //申明
  components: {
    HelloWorld
  }
})
 export default class Home extends Vue {
}
 ```
- props：父组件-》子组件逐级传递数据：子组件定义标签的属性props: ['title']；父组件在使用子组件时传递值给子组件<blog-post title="My journey with Vue"></blog-post>
 - provide/inject：父组件-》子组件直接传递数据
 ```
 父组件
  provide: {
    user: 'John Doe'
  },
 子组件：
 inject: ['user'],
 ```
- 自定义事件：子组件-》父组件 事件通知；
 ```
 父组件监听事件：
 <blog-post ... @enlarge-text="postFontSize += 0.1"></blog-post>
 <blog-post ... @enlarge-text="postFontSize += $event"></blog-post> //父组件通过$event来获取事件传递的值
 子组件发送事件：
 <button @click="$emit('enlargeText')">
  <button @click="$emit('enlargeText', 0.1)"> //第二个值用来传递事件的值
 ```
- 组件的Data Property： 即组件中的属性变量，$data.xxx可以访问
- methods：定义操作属性值的方法，每次都会重新计算
- computed： 计算属性，必须绑定组件变量使用；因为计算的值会缓存，直到绑定变量的值变化才会更新；支持get和set函数
- 侦听属性/watch：数据变化，执行异步或者耗时的操作时使用：question(newQuestion, oldQuestion) {}
#### class && style
- <div :class="[activeClass, errorClass]"></div>
- <div :style="styleObject"></div>
