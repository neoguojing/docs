# vue3 && typescript

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
- v-on:添加一个事件监听器；v-on:click=
- v-model：实现表单输入（input）和应用状态之间的双向绑定
- v-if/v-for
- v-once：只能初始化一次

#### 模板
- .vue文件中的html代码
- 文本插值： <span>Message: {{ msg }}</span>

#### 组件
- 根组件和其他组件没什么不同
- 每个组件实例都有生命周期钩子函数：  created()，mounted等，可以设置监控等
